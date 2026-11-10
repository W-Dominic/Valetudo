# CLAUDE.md

Personal fork of Hypfer/Valetudo (`W-Dominic/Valetudo`). Used to study the code running on the user's Dreame L10s Ultra. Robot details, backups, audit findings and open items: `HANDOFF.md`.

## Repo
npm workspaces: `backend`, `frontend`, `docs`. Node >= 20. Run `npm ci` at the root first (also applies `patches/` via patch-package).

```
npm run lint_all                       # eslint, frontend + backend
npm run ts-check_all                   # tsc --noEmit (JSDoc types in backend)
npm test --workspace=backend           # node --test "test/**/*_test.js"
npm run prepare_commit --workspace=backend    # lint_fix + ts-check + test
npm run build_aarch64 --workspace=backend     # binary for the L10s Ultra, output build/aarch64/valetudo
```

Local dev without a robot: set `robot.implementation` to `MockValetudoRobot` in `local/valetudo_config.json` (gitignored), run `npm run start:dev --workspace=backend`, then `npm run watch --workspace=frontend`.

## Backend layout (`backend/lib`)
- `Valetudo.js`: entry point, wires everything together.
- `core/ValetudoRobotFactory.js`: picks the robot class; with `implementation: "auto"` it calls each class's `IMPLEMENTATION_AUTO_DETECTION_HANDLER()`.
- `core/ValetudoRobot.js`, `core/capabilities/`: generic robot and capability base classes.
- `entities/`: state attributes and map entities exposed by the API.
- `robots/<vendor>/`: vendor implementations; `capabilities/` holds one file per feature.
- `miio/`: miio protocol and the Dummycloud (UDP `127.0.0.1:8053`) that replaces the vendor cloud.
- `webserver/`: Express API under `/api/v2/`; `capabilityRouters/` maps capabilities to routes.
- `mqtt/`: MQTT + Home Assistant autodiscovery.
- `res/default_config.json`: defaults (NTP, updater, webserver, etc.).

## L10s Ultra code path
`robots/dreame/DreameL10SUltraValetudoRobot.js` → `DreameGen2LidarValetudoRobot.js` → `DreameGen2ValetudoRobot.js` → `DreameValetudoRobot.js` → `robots/MiioValetudoRobot.js` → `core/ValetudoRobot.js`.

Dreame control is MIoT: properties and actions addressed by `siid`/`piid`/`aiid`, defined in `robots/dreame/DreameMiotServices.js`.

## The live robot
- `192.168.1.170`, Valetudo `2026.10.0` (commit `78267517`). `master` differs only in unrelated models and docs.
- Read-only HTTP: `/api/v2/valetudo/version`, `/api/v2/robot`, `/api/v2/robot/capabilities`, `/api/v2/robot/state/attributes`, `/api/v2/valetudo/log/content`.
- SSH: `ssh -i ~/.ssh/dreame-robot root@192.168.1.170`. Only run read-only commands unless the user explicitly asks otherwise.

## Rules
- Never commit or push; the user does that.
- This fork is on GitHub. Keep personal or robot-identifying data (backups, keys, serials beyond the model prefix) out of tracked files.
- Upstream does not accept contributions (see `CONTRIBUTING.md`). Changes here are for the user's own use.
- Follow upstream code style: eslint config in `.eslintrc.json` and the generated `.automated`/`.pedantic` overrides; requires sorted (`sort-requires`), JSDoc types in the backend.
