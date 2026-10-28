# Project Rules

## Scope and Git

- Keep changes within the request. Treat review, tests, host changes, commits,
  pushes and merges as separate actions.
- Inspect the current working tree and preserve unrelated changes. Leave review
  edits unstaged unless staging is requested.
- Work on a `codex/<topic>` feature branch. Confirm a new branch name before
  creating it. Never commit or push without explicit human authorization.
- Never commit credentials, private inventory or raw host logs.

## Fan Control

- Keep the role reusable; put host-specific settings in inventory.
- Read temperatures, RPM, PWM and BMC events before changing fan control.
- Never lower hardware alarm thresholds or disable thermal protections to hide a fault.
- Check script syntax before deployment. Verify real fans after deployment.

## Documentation

- Write concise, plain instructions. Remove repeated reminders, development
  history, reasoning diaries and ASCII decoration.
- Keep manual commands usable without Ansible inventory. Show the editable
  configuration and its path before commands that depend on it; put Ansible
  configuration in a separate section.
- Explain values with short YAML or Bash comments. Put separate commands on
  separate lines; do not chain them with `&&`. State the output that confirms success.
- Distinguish role defaults, node2 configuration and measurements. Do not
  present an untested setting or fan speed as verified.
- `README.md` owns the role overview, default curve, deployment, operation,
  controller safety and checks.
- `docs/X12SCZ-F.md` owns node2 wiring, airflow, curve table, measured RPM,
  fan-specific limits and board/case references.
- `inventory/host_vars/node2.example.yml` owns published node2 settings.
  Link to it before deploy commands; do not copy its full curve into README.
- Link across documents instead of repeating host-specific tables and evidence.
- Keep graphics to schematics with short descriptive labels. Put tables,
  parameter lists and large text blocks in Markdown beside the image.
- Include `--diff` in every `ansible-playbook` example or instruction.
