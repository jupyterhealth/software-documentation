---
title: JupyterHealth Software Documentation
#subtitle: ''
---

# Introduction

A JupyterHealth deployment can be characterized as a JupyterHub with the ability to integrate with a CommonHealth CloudStorage service.

The first such deployment will involve a basic JupyterHub configured with GitHub authentication. The CommonHealth server will provide synthetic data.

In later deployments, the JupyterHub will authenticate users against a mock institutional identity provider while the CommonHealth server will provide data collected from an internal Friends and Family Study.

The long term goal is for an organization to be able to deploy a JupyterHub within their existing infrastructure and accessible through existing platforms.

## Development

Development is discussed in:

- a weekly Zoom meeting
- the private channel `#jupyter-health-software` on 2i2c's Slack instance
- the [jupyterhealth/jupyter-health-software](https://github.com/jupyterhealth/jupyter-health-software) github repository
