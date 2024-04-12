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

The image specifying the user environment is deployed from the [jupyterhealth/singleuser-image](https://github.com/jupyterhealth/singleuser-image) repo
and hosted at https://quay.io/repository/jupyterhealth/singleuser-premvp.

The user image is based on the `jupyter/scipy-notebook` image.

The user image also contains a `jupyter_health` package (not yet published as a package available elsewhere),
which provides a `JupyterHealthCHClient` class, which loads credentials from the environment, so needs no arguments:

```python
from jupyter_health import JupyterHealthCHClient

ch_client = JupyterHealthCHClient()
```

### Registering new patients

New patients can be registered with deep links:

```python
deep_link = ch_client.construct_authorization_request_deeplink(
    "patient-id",
    scope="OMHealthResource.HeartRate.Read OMHealthResource.BloodPressure.Read",
    expiration_seconds=2592000,
)
print(deep_link)
```

The resulting URL can be shared with patients to start uploading data to CommonHealth Cloud.

### Fetching patient data

Once a patient is enrolled and has uploaded some data,
patient data can be fetched using the patient id/name used when creating the deep link:

```python
records = ch_client.fetch_data("patient-id")
```

Each record is a `ResourceHolder` instance, where `json_content` contains the actual data.
`json_content` has a `header` with metadata identifying the schema, and a `body` containing the measurement itself.
To extract blood pressure measurements:

```python
bp_readings = [
    record.json_content["body"]
    for record in records
    if record.resource_type == "BLOOD_PRESSURE"
]
```

Which will select the data according to the openmhealth blood pressure schema.
These readings can then be passed to a plotting utility like [this one](./examples/openmhealth-bp).
