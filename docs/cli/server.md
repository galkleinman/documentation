---
id: server
title: Temporal CLI server command reference
sidebar_label: server
description: Effectively manage the Temporal Server by using the Temporal command-line interface (CLI). Explore documentation about administration and configuration.
toc_max_heading_level: 4
keywords:
- cli reference
- command-line-interface-cli
- server
- server start-dev
- temporal cli
tags:
- cli-reference
- command-line-interface-cli
- server
- server-start-dev
- temporal-cli
---

<!-- THIS FILE IS GENERATED. DO NOT EDIT THIS FILE DIRECTLY -->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Server commands allow you to start and manage the [Temporal Server](/clusters#temporal-server) from the command line.

Currently, `cli` server functionality extends to starting the Server.

## start-dev

The `temporal server start-dev` command starts a local [Temporal Server](/clusters#temporal-server).
You can access the Web UI at http://localhost:8233.
The default Frontend Service gRPC port used as a target endpoint for client calls is 7233.

Use the following options to change the behavior of this command.

- [--db-filename](/cli/cmd-options#db-filename)

- [--dynamic-config-value](/cli/cmd-options#dynamic-config-value)

- [--headless](/cli/cmd-options#headless)

- [--ip](/cli/cmd-options#ip)

- [--log-format](/cli/cmd-options#log-format)

- [--log-level](/cli/cmd-options#log-level)

- [--metrics-port](/cli/cmd-options#metrics-port)

- [--namespace](/cli/cmd-options#namespace)

- [--port](/cli/cmd-options#port)

- [--sqlite-pragma](/cli/cmd-options#sqlite-pragma)

- [--ui-asset-path](/cli/cmd-options#ui-asset-path)

- [--ui-codec-endpoint](/cli/cmd-options#ui-codec-endpoint)

- [--ui-ip](/cli/cmd-options#ui-ip)

- [--ui-port](/cli/cmd-options#ui-port)

