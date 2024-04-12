---
title: Preliminary Blood Pressure Prototype
subtitle: 'Development: "Pre MVP"'
---

# Proof of Concept

The first instance of a JupyterHealth deployment is a proof-of-concept, or "pre PVM" product. We want to create something that demonstrates the overal architecture can work, but without using real patient data and without requiring access to existing institutional infrastructure.

(commonhealth)=

## CommonHealth

- [Architecture](https://drive.google.com/file/d/1JupkAcP1CHlhOW6qnQib5s29lqOWm-8U/view)
- [CloudStorage Service](https://docs.google.com/document/d/1E7t16ok_mUHA5hYVsTQA5MbCruX0DS_Rqsq37dZ48fs/edit?usp=drive_link)

(jupyterhub)=

## JupyterHub

The pre MVP hub is deployed by 2i2c using their [Managed JupyterHubs Service](https://infrastructure.2i2c.org). The configuration for it is in the [2i2c infrastructure repository](https://github.com/2i2c-org/infrastructure/tree/main/config/clusters/jupyter-health).
