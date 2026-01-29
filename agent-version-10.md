# PMS-OBUSPA v10.0.0 – Complete Configuration Guide

## Table of Contents

1. [Roles and Permissions](#roles-and-permissions)
   - [Overview](#1-overview)
   - [Example 1: Blacklisting a Parameter](#2-example-1-blacklisting-a-parameter)
   - [Example 2: Read-Only Access](#3-example-2-new-controller-with-read-only-access)
2. [USP Services](#usp-services)
   - [Introduction](#4-introduction)
   - [Architecture Overview](#5-architecture-overview)
   - [Environment Setup](#6-preparing-the-environment)
   - [Broker Configuration](#7-configuring-the-usp-broker)
   - [Data Model Provider](#8-implementing-a-usp-service-data-model-provider)
   - [Controller Service](#9-implementing-a-usp-service-as-a-controller)
   - [Notifications and Subscriptions](#10-notifications-and-subscriptions)
   - [Asynchronous Operations](#11-asynchronous-operations)
   - [Validation and Best Practices](#12-validation-and-best-practices)

---

# Roles and Permissions

## 1. Overview

In the **User Services Platform (USP)** protocol (Broadband Forum TR-369), access control is handled through the **ControllerTrust** subtree.

Each connected **Controller** receives permissions via a **Role**, which specifies allowed operations (Read, Write, Add, Delete, Execute, Notify, etc.) on targeted parts of the Agent's data model.

### Key Parameters

For each Controller instance (`Device.LocalAgent.Controller.{i}.`):

- **`InheritedRole`** (Read-only)  
  Automatically assigned based on the secure MTP connection (e.g., TLS client certificate subject, MQTT credentials, or STOMP chain-of-trust). Points to an instance like `Device.LocalAgent.ControllerTrust.Role.1.`

- **`AssignedRole`** (Writable)  
  Explicitly sets a specific Role instance (e.g., `Device.LocalAgent.ControllerTrust.Role.4.`).  
  **If AssignedRole is set, it overrides InheritedRole** during permission checks.

### Permission Location

Permissions are configured at:  
**`Device.LocalAgent.ControllerTrust.Role.{i}.Permission.{j}.`**

---

## 2. Example 1: Blacklisting a Parameter

**Goal**: Block Controller 1 from reading or writing `Device.WiFi.SSID.{i}.SSID` (hide it from Get responses), while allowing access to other SSID parameters (Enable, Stats, etc.).

### Steps

Assuming Role.1 is active via InheritedRole or AssignedRole:

#### Step 1: Verify Current Role

```bash
obuspa -c get Device.LocalAgent.Controller.
```

Example output:
```
Device.LocalAgent.Controller.1.InheritedRole => Device.LocalAgent.ControllerTrust.Role.1
```

#### Step 2: Add New Permission Rule

```bash
obuspa -c add Device.LocalAgent.ControllerTrust.Role.1.Permission
```

Returns: `Added Device.LocalAgent.ControllerTrust.Role.1.Permission.2`

#### Step 3: Configure as Deny Rule

```bash
obuspa -c set Device.LocalAgent.ControllerTrust.Role.1.Permission.2.Targets "Device.WiFi.SSID.{i}.SSID"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.1.Permission.2.Order 2
obuspa -c set Device.LocalAgent.ControllerTrust.Role.1.Permission.2.Param "----"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.1.Permission.2.Obj "----"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.1.Permission.2.Enable true
```

### Result

A `Get Device.WiFi.SSID.{i}.` from this Controller returns stats and other params, but omits SSID:

```json
{
  "Status": "Up",
  "Stats": {
    "PacketsSent": "3280",
    "PacketsReceived": "4846"
  },
  "Name": "cfg033579",
  "MACAddress": "64:64:4A:EE:55:1B",
  "LowerLayers": "Device.WiFi.Radio.2",
  "Enable": "1",
  "BSSID": "64:64:4A:EE:55:1B",
  "Alias": "cpe-1"
}
```

---

## 3. Example 2: New Controller with Read-Only Access

**Goal**: Create a restricted "provisioning" Controller that can read `Device.DeviceInfo.ProvisioningCode` but cannot write it.

### Steps

#### Step 1: Add New Controller

```bash
obuspa -c add Device.LocalAgent.Controller.
```

Returns: `Device.LocalAgent.Controller.2.`

#### Step 2: Enable and Identify Controller

```bash
obuspa -c set Device.LocalAgent.Controller.2.Enable true
obuspa -c set Device.LocalAgent.Controller.2.EndpointID "provisioning-controller-001"
```

#### Step 3: Create Dedicated Role

```bash
obuspa -c add Device.LocalAgent.ControllerTrust.Role.
```

Returns: `Device.LocalAgent.ControllerTrust.Role.4.`

#### Step 4: Configure Role

```bash
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Enable true
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Name "read_only_provisioning"
```

#### Step 5: Add and Configure Permission

```bash
obuspa -c add Device.LocalAgent.ControllerTrust.Role.4.Permission.
```

Returns: `Permission.1`

```bash
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Permission.1.Targets "Device.DeviceInfo.ProvisioningCode"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Permission.1.Order 1
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Permission.1.Param "r---"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Permission.1.Obj "r---"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Permission.1.CmdEv "----"
obuspa -c set Device.LocalAgent.ControllerTrust.Role.4.Permission.1.Enable true
```

#### Step 6: Assign Role to Controller

```bash
obuspa -c set Device.LocalAgent.Controller.2.AssignedRole "Device.LocalAgent.ControllerTrust.Role.4."
```

### Expected Behavior

- `Get Device.DeviceInfo.ProvisioningCode` → succeeds
- `Set Device.DeviceInfo.ProvisioningCode "newvalue"` → fails (access denied error)

---

# USP Services

## 4. Introduction

This section explains how to **configure, run, and test USP Services using OBUSPA** in an embedded Linux / OpenWrt environment. It is intended for customers integrating vendor-specific applications with a USP Agent using **USP Internal Services (Software Modularization)** as defined by the Broadband Forum.

### Goal

Enable clean separation between:

- The **USP Broker** (main OBUSPA instance)
- One or more **USP Services** (modular applications)

### Service Capabilities

Each USP Service may:

- Provide a **data model** (Agent role)
- Act as a **Controller** of other services
- Or perform **both roles simultaneously**

---

## 5. Architecture Overview

### 5.1 Key Components

| Component                | Description                                                          |
|--------------------------|----------------------------------------------------------------------|
| USP Broker               | Main OBUSPA instance; aggregates data models and routes USP messages |
| USP Service              | Secondary OBUSPA instance or vendor application                      |
| UDS (Unix Domain Socket) | IPC mechanism between Broker and Services                            |
| Vendor Backend           | C code implementing service logic                                    |

### 5.2 Logical Roles

| Role                        | Description                                  |
|-----------------------------|----------------------------------------------|
| Agent / Data Model Provider | Exposes objects and parameters to the Broker |
| Controller                  | Sends USP GET/SET/OPERATE requests           |

> **Note**: A USP Service may act as **Agent**, **Controller**, or **both**.

---

## 6. Preparing the Environment

### 6.1 Build Requirements

- OBUSPA built with **Broker support enabled** (default)
- Unix Domain Socket support
- Optional: readline library for CLI-based controllers

### 6.2 Broker Support Flag

Broker support can be disabled by defining:

```c
#define REMOVE_USP_BROKER
```

**Important**: Do **not** define this when using USP Services.

---

## 7. Configuring the USP Broker

The Broker is the central coordination point. It must expose **two UDS endpoints**:

1. **Controller Socket** – used by Services acting as Agents
2. **Agent Socket** – used by Services acting as Controllers

### 7.1 Broker Factory Reset Configuration

Example `broker_reset.txt` additions:

```text
Device.UnixDomainSockets.UnixDomainSocket.1.Alias "cpe-1"
Device.UnixDomainSockets.UnixDomainSocket.1.Mode "Listen"
Device.UnixDomainSockets.UnixDomainSocket.1.Path "/var/run/usp/broker_controller_path"

Device.UnixDomainSockets.UnixDomainSocket.2.Alias "cpe-2"
Device.UnixDomainSockets.UnixDomainSocket.2.Mode "Listen"
Device.UnixDomainSockets.UnixDomainSocket.2.Path "/var/run/usp/broker_agent_path"
```

**Note**: `broker_reset.txt` will also contain the MTP configuration.

### 7.2 Starting the Broker

```bash
obuspa -f /var/obuspa/usp_broker.db -s /tmp/broker_cli -p -v 4 -r broker_reset.txt -i eth0
```

### 7.3 Verify Broker Operation

```bash
obuspa -s /tmp/broker_cli -c get Device.
```

---

## 8. Implementing a USP Service – Data Model Provider

### 8.1 Purpose

A Data Model Provider exposes vendor functionality (parameters and operations) to the Broker.

### 8.2 Service Configuration

Each service must:
- Use a **unique EndpointID**
- Connect to the **Broker Controller socket**

Example `service1_reset.txt`:

```text
Device.LocalAgent.EndpointID "proto::service1"

Device.UnixDomainSockets.UnixDomainSocket.1.Alias "cpe-1"
Device.UnixDomainSockets.UnixDomainSocket.1.Mode "Connect"
Device.UnixDomainSockets.UnixDomainSocket.1.Path "/var/run/usp/broker_controller_path"
```

### 8.3 Vendor Backend Implementation

Vendor code is implemented in:

```
src/vendor/vendor.c
```

Register objects **only when running as a USP Service**:

```c
#include "common_defs.h"

int NotifyChange_ParamA(dm_req_t *req, char *value)
{
    USP_LOG_Info("ParamA changed to %s", value);
    return USP_ERR_OK;
}

int VENDOR_Init(void)
{
    if (RUNNING_AS_USP_SERVICE())
    {
        USP_REGISTER_Object("Device.Test.ServiceA", NULL, NULL, NULL, NULL, NULL, NULL);
        USP_REGISTER_DBParam_ReadWrite(
            "Device.Test.ServiceA.ParamA",
            "default_A",
            NULL,
            NotifyChange_ParamA,
            DM_STRING);
    }
    return USP_ERR_OK;
}
```

### 8.4 Starting the Service

```bash
obuspa -f /usr/local/var/obuspa/usp_service1.db -s /tmp/service1_cli -p -v 4 -r service1_reset.txt -i eth0 -R "Device.Test"
```

The `-R` option defines which part of the data model is registered with the Broker.

### 8.5 Testing from Broker CLI

#### Get Service Parameters

```bash
obuspa -s /tmp/broker_cli -c get Device.Test.
```

#### Set a Parameter

```bash
obuspa -s /tmp/broker_cli -c set Device.Test.ServiceA.ParamA new_value
```

Verify service logs to confirm callback execution.

---

## 9. Implementing a USP Service as a Controller

### 9.1 Purpose

A Controller Service issues USP commands to:
- Other USP Services
- The Broker's own Agent data model

**Typical use cases**:
- Orchestration
- Policy engines
- Diagnostics controllers

### 9.2 Controller Socket Configuration

Service must connect to the **Broker Agent socket**, create service2_reset.txt that contains:

```text
Device.LocalAgent.EndpointID "proto::service2"
Device.UnixDomainSockets.UnixDomainSocket.2.Alias "cpe-2"
Device.UnixDomainSockets.UnixDomainSocket.2.Mode "Connect"
Device.UnixDomainSockets.UnixDomainSocket.2.Path "/var/run/usp/broker_agent_path"
```

### 9.3 Controller API

**Available calls**:
- `USP_SERVICE_Get()`
- `USP_SERVICE_Set()`
- `USP_SERVICE_Add()`
- `USP_SERVICE_Delete()`
- `USP_SERVICE_Operate()`
- `USP_SERVICE_RegisterNotificationCallback()`

All calls are **blocking** with timeout protection.

### 9.4 Example: CLI-Based Controller

- Create a worker thread in `VENDOR_Start()`
- Parse user input
- Convert commands to `USP_SERVICE_*` calls

This allows interactive testing without an external Controller.

### 9.5 Vendor Backend Controller Implementation
In vendor.c adapt VENDOR_Start() to create a controller thread:

```c
int VENDOR_Start(void)
{
    int err;

    err = OS_UTILS_CreateThread("ctrler", ControllerThread, NULL);
    if (err != USP_ERR_OK)
    {
        USP_LOG_Error("%s: Failed to create controller thread (err=%d)", __FUNCTION__, err);
        return err;
    }

    return USP_ERR_OK;
}
```

In the same file add the controller thread:
```c
#include "usp_service.h"
#include "usp_api.h"
#include "kv_vector.h"
#include "text_utils.h"
#include <stdio.h>
#include <readline/readline.h>
#include <readline/history.h>

void *ControllerThread(void *args)
{
   str_vector_t sv_params;
   kv_vector_t kvv_params;
   int index = 0;
   char errMsg[128];
   kv_pair_t *kv;
   char *s = NULL;
   int err;
   int instance;
   using_history();
   s = readline(">>");
   while (s != NULL)
   {
        if (strcmp(s, "quit") == 0)
        {
            free(s);
            break;
        }
        if (*s == '\0')
        {
            goto next;
        }
        add_history(s);
        errMsg[0] = '\0';
        USP_ARG_Init(&kvv_params);
        USP_STR_VEC_Init(&sv_params);
        TEXT_UTILS_SplitString(s, &sv_params, ",");
        #define VENDOR_TEST_USP_TIMEOUT 30
        if (strcmp(sv_params.vector[0], "GET")==0)
        {
            for (index = 1 ; index < sv_params.num_entries ; index++)
            {
                USP_ARG_Add(&kvv_params, sv_params.vector[index], NULL);
            }
            err = USP_SERVICE_Get(&kvv_params, VENDOR_TEST_USP_TIMEOUT, errMsg, sizeof(errMsg));
        }
        else if (strcmp(sv_params.vector[0], "SET")==0)
        {
            for (index = 1 ; index < sv_params.num_entries ; index+=2)
            {
                USP_ARG_Add(&kvv_params, sv_params.vector[index], sv_params.vector[index+1]);
            }
            err = USP_SERVICE_Set(&kvv_params, VENDOR_TEST_USP_TIMEOUT, errMsg, sizeof(errMsg));
        }
        else if (strcmp(sv_params.vector[0], "ADD")==0)
        {
            if (sv_params.num_entries < 2)
            {
                USP_LOG_Error("ADD command requires object path");
                goto next;
            }
            for (index = 2 ; index < sv_params.num_entries ; index+=2)
            {
                USP_ARG_Add(&kvv_params, sv_params.vector[index], sv_params.vector[index+1]);
            }
            err = USP_SERVICE_Add(sv_params.vector[1], &kvv_params, VENDOR_TEST_USP_TIMEOUT, &instance, errMsg, sizeof(errMsg));
            if (err == USP_ERR_OK)
            {
                printf("Created instance: %d\n", instance);
            }
        }
        else if (strcmp(sv_params.vector[0], "DELETE")==0)
        {
            if (sv_params.num_entries < 2)
            {
                USP_LOG_Error("DELETE command requires object path");
                goto next;
            }
            err = USP_SERVICE_Delete(sv_params.vector[1], VENDOR_TEST_USP_TIMEOUT, errMsg, sizeof(errMsg));
        }
        else
        {
            USP_LOG_Error("Unrecognised command %s", sv_params.vector[0]);
            goto next;
        }
        
        if (err != USP_ERR_OK)
        {
            printf("Error: %s\n", errMsg);
        }
        
        for (index = 0 ; index < kvv_params.num_entries ; index++)
        {
            kv = &kvv_params.vector[index];
            assert(kv->value != NULL);
            printf("\"%s\" => \"%s\" \n", kv->key, kv->value);
        }
next:
        USP_ARG_Destroy(&kvv_params);
        USP_STR_VEC_Destroy(&sv_params);
        free (s);
        s = readline(">>");
    }
    return NULL;
}
```

In usp_service.h include:

```c
#include "usp-msg.pb-c.h"
#include "device.h"
```

Also we need to add the -lreadline to the makefile.am:

```
obuspa_LDADD = -lm -ldl -lpthread -lrt -lreadline
```

And add +libreadline in the following file openwrt/obuspa/Makefile:

```
DEPENDS:=+libcoap +libopenssl +libcurl +libsqlite3 +libmosquitto +libjson-c +libblobmsg-json +libwebsockets-openssl +libreadline
```

### 9.6 Starting the Service

```bash
obuspa -f /usr/local/var/obuspa/usp_service2.db -s /tmp/service2_cli -p -v 4 -r service2_reset.txt -i eth0 -R ""
```

### 9.7 Testing from Service as Controller

#### Get Parameters

```bash
>>GET,Device.WiFi.
Device.WiFi.AccessPointNumberOfEntries" => "5" 
"Device.WiFi.RadioNumberOfEntries" => "2" 
"Device.WiFi.SSIDNumberOfEntries" => "5"
```

#### Set a Parameter

```bash
SET,Device.WiFi.SSID.1.SSID,MyNetwork
"Device.WiFi.SSID.5.SSID" => "MyNetwork"
```

#### Add object

In this example, we will add a new SSID object to the WiFi object with SSID "NewNetwork"
```bash
ADD,Device.WiFi.SSID.,SSID,NewNetwork
```

#### Delete object

Update instance '2' with the correct SSID:
```bash
DELETE,Device.WiFi.SSID.2.
```

---

## 10. Notifications and Subscriptions

USP Services acting as Controllers may subscribe to notifications.

### 10.1 Creating a Subscription

```c
KV_VECTOR_Add(&kvv_params, "Enable", "true");
KV_VECTOR_Add(&kvv_params, "ID", "NOTIFY-EXAMPLE");
KV_VECTOR_Add(&kvv_params, "NotifType", "ValueChange");
KV_VECTOR_Add(&kvv_params, "ReferenceList", "Device.Time.LocalTimeZone");

USP_SERVICE_Add("Device.LocalAgent.Subscription.",
                &kvv_params,
                10,
                &instance,
                errMsg,
                sizeof(errMsg));
```

### 10.2 Notification Callback

```c
void notify_cb(char *id, char *path, kv_vector_t *args,
               char *cmd_key, int err_code, char *err_msg)
{
    USP_LOG_Info("Notification %s on %s", id, path);
}
```

---

## 11. Asynchronous Operations

Some operations return immediately and complete later.

- Subscribe with `NotifType = OperationComplete`
- Match responses using `cmd_key`

This is required for long-running operations such as uploads or diagnostics.

---

## 12. Validation and Best Practices

### 12.1 Validation Checklist

- ✔ Broker running and listening on both sockets
- ✔ Unique EndpointID per service
- ✔ Correct UDS mode (Listen vs Connect)
- ✔ `-R` option specified when registering data models
- ✔ Service logs show USP handshake and traffic
- ✔ Broker CLI can GET/SET/ADD/DELETE service parameters

### 12.2 Best Practices

- Keep services **small and modular**
- Separate **control logic** from **data ownership**
- Use Controller services for orchestration
- Avoid circular control dependencies
- Log USP interactions at INFO level during development

---

## 13. Summary

### Roles and Permissions

OBUSPA's ControllerTrust mechanism provides fine-grained access control through:
- Role-based permissions
- Configurable allow/deny rules
- Support for both inherited and assigned roles
- Flexible targeting of data model elements

### USP Services

USP Services in OBUSPA enable:
- Clean software modularization
- Independent feature development
- Local service-to-service control
- Full alignment with Broadband Forum USP architecture

This approach scales from simple reboot services to complex, multi-module embedded platforms.

---

**Document Version**: 1.0  
**Last Updated**: January 2026  
**OBUSPA Version**: v10.0.0