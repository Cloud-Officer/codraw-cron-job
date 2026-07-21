# Code Review: codraw/cron-job

Reviewed: all PHP sources, `composer.json`, DI integration, and tests of the
`codraw-cron-job` package (namespace `Draw\Component\CronJob`).

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **M2 (composer.json missing dependencies)** — `composer.json`:
  - Added `symfony/validator: ^6.4.0` to `require` (used by `Entity/CronJob.php`
    constraint attributes; already transitive via `codraw/core`).
  - Added `psr/log: ^3` to `require` (used by
    `EventListener/PostExecutionQueueCronJobListener.php`).
  - Added `doctrine/persistence: ^2.2 || ^3.0` to `require` (`ManagerRegistry` is
    type-hinted in `CronJobProcessor`, both commands, and the listener; previously
    only transitive via `doctrine/orm`).
  - Added `doctrine/collections: ^2.2` to `require` (`ArrayCollection`/`Criteria`/
    `Selectable` used directly in `Entity/CronJob.php`; previously only transitive).
  - Added a `suggest` entry `codraw/framework-extra-bundle: To integrate in Symfony`
    covering the optional `DependencyInjection/` bridge (`codraw/dependency-injection`
    and `symfony/config` usage). This matches the monorepo convention (e.g.
    `codraw-messenger`): every sibling package keeps `codraw/dependency-injection`
    in `require-dev` and suggests the framework bundle, so it was deliberately NOT
    moved to `require`.
- **M1** — `CronJobProcessor.php`: the catch block now guards
  `$process->getOutput()`/`getErrorOutput()` with `$process->isStarted()`, so a
  process that failed to start no longer throws `LogicException` out of the catch
  and leaves the execution stuck in `RUNNING`.
- **M4** — `Message/ExecuteCronJobMessage.php`: `stamp()` now checks
  `instanceof CronJobExecution` before calling `getCronJob()`, removing the
  potential fatal when the property holds a `PropertyReferenceStamp`.
- **L1** — `EventListener/PostExecutionQueueCronJobListener.php`: logger calls use
  `?->`, so an explicitly injected `null` logger can no longer cause a null deref
  (constructor signature left unchanged to avoid breaking existing wiring).
- **L3** — same file: docblock example corrected to
  `--draw-post-execution-queue-cron-job` (doubled `draw-` removed).

### Validation (2026-07-20, second pass)

- `composer install` (CI flags) resolves and installs cleanly with the updated
  `composer.json` — no constraint adjustment was needed.
- `vendor/bin/phpunit`: **OK — 32 tests, 226 assertions, 0 failures.** The 18
  PHPUnit notices ("mock without expectations") are pre-existing test-style
  notices, identical without the fixes (verified via `git stash`).
- PHPStan (`phpstan.dist.neon`): 4 `property.unusedType` errors
  (`Entity/CronJob.php:27`, `Entity/CronJobExecution.php:37`,
  `Message/ExecuteCronJobMessage.php:17` ×2). All four are pre-existing and
  identical without the fixes (verified via `git stash`); they stem from
  properties assigned only by reflection (Doctrine identity generation and the
  messenger Doctrine-reference listener), so the union types are intentional
  and were left untouched.
- `markdownlint-cli2`: 0 errors, nothing to fix.
- No code changes were required in this validation pass; no test expectations
  needed updating.

Not fixed (deliberately, to avoid disrupting consumers): H1 and H2 (change
execution/retry semantics — design decisions), M3 (`symfony/amqp-messenger` left in
`require`; removing/demoting it could break consumers relying on it transitively —
open item), M5 (threat-model documentation decision), M6 (entity/event API change),
L2 (entity API hardening), L4 (compiler-pass design), L5 (repo convention is
`>=8.5`), L6 (output-format change).

## Overall assessment

This is a small, focused component (~10 source classes) that manages database-driven
cron jobs and executes them through Symfony Messenger. The code is clean, modern
(strict types, constructor promotion, attributes), and the core happy paths are well
tested. However, the execution pipeline in `CronJobProcessor::process()` has real
robustness gaps: work that can throw is performed *after* the execution is flushed as
`RUNNING` but *outside* the try/catch, which can leave executions permanently stuck in
`RUNNING` and cause Messenger retries to re-run jobs; there is also no idempotency
guard against redelivered messages. `composer.json` is missing several dependencies
that the source code uses directly, and the message class hard-couples the package to
the AMQP transport. None of these are exploitable security flaws in themselves, but
the "commands live in the database and are run through a shell with container
parameter resolution" design deserves an explicit threat-model note.

## Findings

### High

#### H1. Command parameter resolution can throw outside the try/catch, leaving the execution stuck in `RUNNING`

`CronJobProcessor.php:62-74` — the flow is:

1. `$execution->start(); $manager->flush();` → state persisted as `RUNNING`
2. `$manager->getConnection()->close();`
3. `$this->parameterBag->resolveValue($event->getCommand())` and
   `$this->processFactory->createFromShellCommandLine(...)` — **outside** the
   `try` block that only wraps `mustRun()`.

`ParameterBag::resolveValue()` throws `ParameterNotFoundException` for any
`%token%`-looking substring (regex `%([^%\s]+)%`). Shell commands routinely contain
percent signs — e.g. `date +%Y-%m-%d`, `curl --data-urlencode a%3Db`, `printf '%s'`.
The `%Y-%` portion of `date +%Y-%m-%d` is parsed as a parameter named `Y-` and throws.
Because this happens after the `RUNNING` flush and outside the catch:

- the execution row stays `RUNNING` forever (no `fail()`/`flush()` runs);
- the exception propagates to the Messenger handler, so the transport retries the
  message, which re-runs `process()` (see H2) and fails again.

Users must know to escape `%` as `%%`, which is undocumented. Fix: wrap resolution and
process creation in the same try/catch that records failure, and document `%%`
escaping (or make parameter resolution opt-in).

#### H2. No idempotency/state guard in `process()` — retried or duplicated messages re-execute the job

`CronJobProcessor.php:46` — the only gate before running the shell command is
`$execution->isExecutable(...)` (`Entity/CronJobExecution.php:179-196`), which checks
`active`/`force` and time-to-live but **not the execution state**. Any redelivered
`ExecuteCronJobMessage` — Messenger retry after a handler exception (e.g. H1, or a
`flush()` failure at `CronJobProcessor.php:92` after the connection was closed), a
transport redelivery, or an operator re-dispatching — will run the command a second
time even when the execution is already `RUNNING`, `TERMINATED`, or `ERRORED`. For
non-idempotent jobs (billing, mail-outs, purges) this is a double-execution bug.
Fix: refuse (skip) executions whose state is not `STATE_REQUESTED`, ideally with a
DB-level compare-and-swap (`UPDATE ... SET state='running' WHERE id=? AND
state='requested'`) so concurrent workers cannot both start the same execution.

### Medium

#### **[FIXED]** M1. The catch block itself can throw when the process failed to start

`CronJobProcessor.php:80-90` — the `catch (\Throwable)` calls `$process->getOutput()`
and `$process->getErrorOutput()`. If `mustRun()` failed *before* the process started
(`proc_open` failure, invalid working dir → `RuntimeException` from
`Process::start()`), Symfony `Process::getOutput()` throws a `LogicException`
("Process must be started before calling getOutput()"). That exception escapes the
catch, `fail()`/`flush()` never run, and the execution is again stuck in `RUNNING`
(same failure mode as H1). Guard with `$process->isStarted()` before reading output.

#### **[FIXED]** M2. `composer.json` is missing dependencies the source uses directly

`composer.json:18-30` —

- `Entity/CronJob.php:14` uses `Symfony\Component\Validator\Constraints` attributes,
  but `symfony/validator` is neither required nor suggested. In an app without the
  validator the constraints silently do nothing (and reflection instantiation of the
  attributes would fatal if anything does read them).
- `DependencyInjection/CronJobIntegration.php:14` type-hints
  `Symfony\Component\Config\Definition\Builder\ArrayNodeDefinition`, but
  `symfony/config` is not required.
- `EventListener/PostExecutionQueueCronJobListener.php:8-9` uses `psr/log`
  (only transitively available).
- `DependencyInjection/CronJobIntegration.php` and
  `DependencyInjection/Compiler/AddPostCronJobExecutionOptionPass.php:6` depend on
  `Draw\Component\DependencyInjection\*` classes from `codraw/dependency-injection`,
  which is only in `require-dev` (`composer.json:33`). Any consumer that autoload-scans
  or instantiates the integration without the framework meta-package gets a fatal
  "class not found". These classes should be behind an explicit require or a suggest
  with a class_exists guard.

#### M3. Hard coupling to the AMQP transport

`composer.json:25` requires `symfony/amqp-messenger` unconditionally, solely because
`Message/ExecuteCronJobMessage.php:12,40-44` applies an `AmqpStamp` for priority.
Users on Doctrine/Redis/SQS transports are forced to install the AMQP bridge (which
needs `ext-amqp` at runtime for actual use), and job `priority` silently does nothing
on those transports. Consider making the stamp conditional
(`class_exists(AmqpStamp::class)`) and moving the bridge to `suggest`, or delegating
priority stamping to a listener in the AMQP-specific layer.

#### **[FIXED]** M4. `ExecuteCronJobMessage::stamp()` can fatal on a restored message

`Message/ExecuteCronJobMessage.php:17,40` — `$execution` is typed
`PropertyReferenceStamp|CronJobExecution|null`, but `stamp()` calls
`$this->execution?->getCronJob()`. `?->` only guards `null`; if `stamp()` is ever
invoked while the property holds a `PropertyReferenceStamp` (message re-dispatched
after the Doctrine-reference envelope listener replaced the property), this is a fatal
"call to undefined method". An `instanceof CronJobExecution` check would make the
union safe (and would satisfy static analysis — the empty
`phpstan-baseline.neon` suggests this slips through the current level).

#### M5. Threat model: database rows are shell commands, with container parameter resolution

`CronJobProcessor.php:69-74` runs `CronJob::$command` through
`createFromShellCommandLine()` — i.e., a real shell — after resolving `%param%`
placeholders against the *container parameter bag*. This is the package's purpose, but
two consequences should be documented prominently:

- Anyone with write access to the `cron_job__cron_job` table (e.g. a Sonata admin
  user, or SQL injection elsewhere in the app) gets arbitrary shell execution on the
  worker host. Cron-job administration is effectively RCE-equivalent privilege.
- Parameter resolution lets such a user read any container parameter
  (e.g. `echo %kernel.secret%`, `%env(DATABASE_URL)%`-style params), so secrets can be
  exfiltrated through job output/error records without even running a malicious
  binary.

There is no allow-list or validation of the command. At minimum the README should
state the trust assumption; a stronger design would restrict resolution to an explicit
allow-list of parameters.

#### M6. `PreCronJobExecutionEvent` fatals on a `CronJob` with a null command

`Event/PreCronJobExecutionEvent.php:12,18` — `$this->command` is non-nullable
`string`, assigned from `CronJob::getCommand()` which returns `?string`
(`Entity/CronJob.php:111`). A `CronJob` persisted without a command (the property is
nullable in PHP even though the column is `nullable: false` and only a validator
constraint — which may not be installed, see M2 — guards it) produces a `TypeError`
inside `CronJobProcessor::process()` at line 53, after nothing has been flushed as
skipped/failed. Edge case, but the entity API makes it reachable.

### Low

#### **[FIXED]** L1. Nullable logger with non-null default invites a null deref

`EventListener/PostExecutionQueueCronJobListener.php:27` — `private ?LoggerInterface
$logger = new NullLogger()`. The type admits `null`, and `triggerCronJob()` calls
`$this->logger->error(...)` unguarded (lines 54, 59). If DI wiring ever passes an
explicit `null` (e.g. `?logger` service reference), this fatals. Drop the `?` (the
default already provides the null-object).

#### L2. `CronJobExecution::end()` null-deref when called before `start()`

`Entity/CronJobExecution.php:214` — `$this->getExecutionStartedAt()->getTimestamp()`
assumes `start()` ran. `end()` and `fail()` are public; calling either on a fresh
execution fatals. Internal usage is safe today, but a guard or non-nullable design
would harden the entity API.

#### **[FIXED]** L3. Docblock example uses a wrong option name

`EventListener/PostExecutionQueueCronJobListener.php:18` — the example says
`--draw-draw-post-execution-queue-cron-job` (doubled `draw-`); the actual option is
`draw-post-execution-queue-cron-job` (line 22).

#### L4. Compiler pass can break commands that already define the option

`DependencyInjection/Compiler/AddPostCronJobExecutionOptionPass.php:22-34` — an
`addOption()` method call is appended to *every* console command definition. If any
command already defines an option (or shortcut) with that name, `addOption()` throws a
`LogicException` at service instantiation, breaking that command entirely. The
namespaced option name makes collision unlikely, but the pass could check the
definition's existing calls or the command could ignore the failure.

#### L5. `composer.json` platform constraint is unusual

`composer.json:19` — `"php": ">=8.5"` is an open-ended lower bound (future PHP 9
allowed implicitly) combined with Symfony `^6.4`, whose official platform testing
predates PHP 8.5. A caret constraint (`^8.5`) and/or Symfony `^6.4 || ^7.x` alignment
with the rest of the framework would be safer.

#### L6. Unbounded process output stored into the `error` column

`CronJobProcessor.php:81-89` / `Entity/CronJobExecution.php:60` — the full stdout and
stderr of a failed process are concatenated into one `text` column. A chatty failing
job (megabytes of output) bloats the row, and output may include secrets echoed by the
command. Consider truncating to a sane length.

## Strengths

- **Clean, modern PHP**: strict types everywhere, constructor property promotion,
  readonly-style value flow, attributes for ORM/DI/Messenger/AsCommand — no deprecated
  Symfony API usage found.
- **Good separation of concerns**: queueing (`queue()`), execution (`process()`),
  scheduling check (`CronJob::isDue()`), and extension points (pre/post events with a
  mutable command and a cancellation flag) are cleanly separated; the
  `PreCronJobExecutionEvent` command-override hook is a nice extensibility point.
- **Sensible operational touches**: closing the DBAL connection before running
  long-lived child processes (`CronJobProcessor.php:65-67`) avoids "server has gone
  away" issues; `EXTRA_LAZY` + `Criteria`-based `getRecentExecutions()` avoids loading
  the whole executions collection; a DB index on `state`.
- **Schedule input is normalized/validated** at the setter
  (`Entity/CronJob.php:128-137`) via `CronExpression`, so invalid cron strings are
  rejected before persistence.
- **Entity state machine is encapsulated**: state mutators (`setState`, timestamps,
  exit code) are private; transitions only via `start()/end()/fail()/skip()/
  acknowledge()`.
- **Empty phpstan baseline** — the package carries no suppressed static-analysis debt.

## Test coverage

Qualitatively good for the core, with clear gaps around the periphery:

- **Well covered**: `CronJobProcessor` (queue, success, failure, inactive-skip,
  cancelled-skip, parameter resolution, timeout pass-through — `Tests/CronJobProcessorTest.php`),
  `CronJobExecution::isExecutable()` matrix incl. TTL and force
  (`Tests/Entity/CronJobExecutionTest.php`), both console commands incl. not-found and
  not-due paths (`Tests/Command/*`), the message handler delegation
  (`Tests/MessageHandler/ExecuteCronJobMessageHandlerTest.php`), and DI service
  registration (`Tests/DependencyInjection/CronJobIntegrationTest.php`).
- **Not covered**: `PostExecutionQueueCronJobListener` (no test at all),
  `AddPostCronJobExecutionOptionPass`, `ExecuteCronJobMessage::stamp()` (AMQP priority
  stamping), `CronJob` entity behavior (`isDue()`, `setSchedule()` normalization/
  invalid input, `newExecution()`), and the events. Notably, none of the failure modes
  in findings H1/H2/M1 (percent-sign commands, message redelivery, process-start
  failure) have tests — unsurprising, since the bugs exist.
- Tests are unit-level with mocks throughout; there is no integration test exercising
  a real Messenger transport or a real DB, so the connection-close/reconnect behavior
  and Doctrine-reference message round-trip are untested here (possibly covered
  elsewhere in the monorepo).
