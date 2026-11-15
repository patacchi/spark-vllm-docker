# Operation Pointers

Guide: `docs/AGENT_RUNBOOK.md` (authoritative for host prep, recipe selection, cluster discovery, image/model setup, recipe launches, live-server verification). Networking details: `docs/NETWORKING.md` (dual-Spark direct, QSFP/RoCE switch, 3-node mesh, closed rings of 4+, automated setup helper).

Operational actions (only on explicit user request): non-dry recipe run, `--setup`, `--discover`, image build, model download, container launch. Never prune/overwrite/stop/remove existing resources unless the task requires it.

Common quick starts (from README): single node `./run-recipe.sh recipes/<name>.yaml --solo --setup`; cluster: checkout on head node, networking first, then `./run-recipe.sh recipes/<name>.yaml --setup`. Missing model in HF cache → runner stops and asks to rerun with `--setup`.