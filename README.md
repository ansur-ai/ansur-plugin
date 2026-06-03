# Ansur — Claude Code plugin

Turns Claude Code into the **meta-harness**: the build-time agent that drives the
[`@ansur-ai/cli`](https://www.npmjs.com/package/@ansur-ai/cli) to build your AI
employee on the [Ansur](https://useansur.com) platform — set up a tenant, connect
systems, author a bundle, ship it, and wire a channel.

## Install

```
/plugin marketplace add ansur-ai/ansur-plugin
/plugin install ansur@ansur
```

Then start with the onboarding command:

```
/ansur-onboard
```

You'll also need the CLI on your PATH:

```
npm install -g @ansur-ai/cli
ansur login
```

> Login is gated to the Ansur beta — your email must be on the allowlist for
> `ansur login` to succeed.

## What's in it

One skill (`ansur`) — a spine + on-demand reference covering the build loop
(tenant → connectors → bundle → channel) — plus the `/ansur-onboard` entry point.
The plugin ships *behavior*; the `ansur` CLI is the action surface.

## License

See the Ansur platform terms at https://useansur.com.
