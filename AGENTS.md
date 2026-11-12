# AGENTS.md: contract-executor

Instructions for coding agents working in this repository. Read this before changing anything.

## What this is

The Java 11 process that compiles, deploys and runs the smart contracts of the CREDITS blockchain.
Every node starts its own executor and talks to it over Thrift: the node asks it to compile source,
execute a method on a contract state, or read getters; the executor answers with the new state,
the return value and any transactions the contract emits. Contracts run in a sandbox (no files,
network, threads, reflection or class loading).

Related repositories of [CreditsCommunity](https://github.com/CreditsCommunity):

- `node`: starts the executor and pins the release it ships in `packaging/executor-tag`.
- `smart-contract-api`: the classes contracts extend (`sc-api`, `sc-api-internal`).
- `thrift-api-client`: the Thrift clients and shared utilities (`general`, `node_client`, `executor_client`).

## Branches: read this first

- `master` is the upstream CREDITSCOM history (build 1518 plus a README commit). It has no CI and
  no longer builds from source as published.
- The community line is the branch **`build/reproducible-1518`**: a reproducible build of 1518
  (`BUILDING.md`), the CI, the release workflow and the compatible fixes. Pull requests go there;
  the releases `executor-community-N` are tagged on it.

## Platform constraints

- Mainnet runs the official node 4.2.532 with executor build 1518. Every contract result (status,
  return value, new state, emitted transactions) of a community build must be identical to 1518's,
  or nodes running it would disagree with the network. Never change consensus-relevant behaviour
  unilaterally.
- On mainnet a contract cannot call another contract: 1518 fails every `invokeExternalContract`,
  read-only calls included. No value, not even CS, passes from one contract to another: the node
  refuses a contract's `sendTransaction` to a contract. Enabling either changes consensus and needs
  a coordinated, height-gated upgrade of all trusted-capable nodes.
- Amounts in `sendTransaction` are `double`s. A call whose arguments do not match the declared
  types fails, and the fee is still paid.

## Layout (on `build/reproducible-1518`)

| Path | Contents |
|---|---|
| `contract-executor/` | the executor: Thrift server and handler (`com.credits.thrift`), execution and compilation services (`com.credits.service`), sandbox (`com.credits.secure`), node client |
| `sc-api-support/` | the runtime side of the contract API, shipped as `sc-api-support.jar` next to the executor |
| `third-party/mvn-repo`, `third-party/maven-settings.xml` | vendored Maven artifacts; every repository except Maven Central is mirrored there |
| `BUILDING.md` | why 1518 no longer builds as published, and the exact steps |
| `.github/workflows/build.yml` | build, comparison with 1518, release |

## Build and test

JDK 11.0.3 (build 1518 used it; later 11.0.x `javac` versions produce different bytes) and Maven.
The full history is needed: the build number is the git commit count. From `BUILDING.md`:

```sh
export VENDORED_MAVEN_REPO=$PWD/third-party/mvn-repo
MVN="mvn -s $PWD/third-party/maven-settings.xml"
# smart-contract-api and thrift-api-client at the commits pinned in .github/workflows/build.yml
(cd thrift-api-client && $MVN -DskipTests install)
(cd smart-contract-api && $MVN -DskipTests install)
$MVN -DskipTests package
$MVN -pl contract-executor -am test -Dtest=ContractExecutorHandlerTest -DfailIfNoTests=false -Dsurefire.failIfNoSpecifiedTests=false
```

Jars: `contract-executor/target/contract-executor-v0.8.4-b<N>.jar`, `sc-api-support/target/sc-api-support-v0.8.4-b<N>.jar`.

## CI (`build.yml`, on `build/reproducible-1518`)

On every push and pull request:

- `build`: the dependencies and the executor with JDK 11.0.3; fails if anything is fetched from a
  CREDITSCOM repository; compares every class with the jars of mainnet release 4.2.532.0 (reported);
  runs the Thrift handler unit tests.
- `compare-results`: the node's executor harness (`ci/contracts/executor_harness.py`) runs the token
  scenarios on executor 1518 and on this build; every answer must be the same.
- `release`: on tags `executor-*`, publishes `contract-executor.jar`, `sc-api-support.jar` and
  `SHA256SUMS`; `executor-*-rc*` tags are pre-releases. Only maintainers create these tags.

## Conventions

- One topic per pull request, against `build/reproducible-1518`. Commit subjects start with the
  part they change (`executor:`, `sc-api-support:`, `build:`, `ci:`).
- Docs, comments and commit messages in English.
- Never `git add -A` or `git add .`: stage explicit paths. Never delete branches after a merge.

## Pitfalls

- The node accepts an executor only if its build number (git commit count) is within
  `executor_commit_min`..`executor_commit_max` of its `config.ini`; the official config pins
  1518..1518. Commits on top of 1518 raise the count, so a community build needs a higher
  `executor_commit_max`. Squashing or rewriting history changes the count.
- Do not bump dependency versions: `org.eclipse.jdt.core`, the Eclipse bundles and
  `maven-assembly-plugin` are pinned to what 1518 bundled, and thrift-api-client after its libthrift
  0.14 bump breaks `executor_client`'s `module-info`.
- A fix must not change any result the harness compares; add a scenario to the harness when a fix
  touches execution.

## More

- `BUILDING.md`, `third-party/README.md` (on `build/reproducible-1518`)
- The node's `samples/contracts/README.md` (how contracts are written, what the sandbox refuses),
  `docs/tokens.md`, `ci/contracts/`
- Maintainers' agents: a shared Basic Memory knowledge base (project `credits`) holds platform facts,
  procedures and incident notes; use it when your environment provides it.
