# HomeKit NFC Access Seevice Protocol

> **⚠️ Educational Purpose Only**
> This documentation is intended only for educational purposes. All information provided here is based on reverse engineering and research which means information might be incomplete or inaccurate.

## Overview

The NFC Access Service enables the NFC Tap-To-Unlock and Tap-To-Lock features for HomeKit-compatible accessories. This service allows HomeKit controllers to provision NFC-enabled accessories with the necessary cryptographic keys for secure contactless access.

The provisioning process involves three key components:

- **Issuer Key**: Used for authenticating the NFC access system
- **Device Credential Key**: Enables expedited transactions for faster access
- **Reader Key**: Required for NFC reader functionality

After successful provisioning through the NFC Access Service, the accessory becomes ready to support NFC lock features, allowing users to unlock or lock the device by simply tapping their NFC-enabled device (iPhone, Apple Watch, or other compatible devices) against the accessory.

The NFC Access Service operates as a specialized HomeKit service that extends standard lock functionality with contactless capabilities, integrating seamlessly with HomeKit's access control and security framework.

## Services

| Name       | ID  | Description                                        |
| ---------- | --- | -------------------------------------------------- |
| NFC Access | 266 | Enables NFC Tap-To-Unlock and Tap-To-Lock features |

## Characteristics

| Name                               | ID  | Type   | Purpose                                                                | Access     |
| ---------------------------------- | --- | ------ | ---------------------------------------------------------------------- | ---------- |
| Configuration State                | 263 | UINT16 | ¯\_(ツ)\_/¯                                                            | Read       |
| NFC Access Control Point           | 264 | TLV8   | Provisioning and management of issuer / device credential /reader keys | Read/Write |
| NFC Access Supported Configuration | 265 | TLV8   | Configuration and capabilities of NFC access                           | Read       |

## NFC Access Control Point

The primary interface for provisioning and managing NFC access keys. HomeKit controllers use this characteristic to provision the accessory with the necessary cryptographic keys.

### Common enums

#### Key Type

| Value | Meaning |
| ----- | ------- |
| 1     | Ed25519 |
| 2     | NIST256 |

#### Status Code

| Value | Meaning                 |
| ----- | ----------------------- |
| 0     | Success                 |
| 1     | Error: out of resources |
| 2     | Error: duplicate        |
| 3     | Error: does not exist   |
| 4     | Error: not supported    |

### Characteristic root TLV8

| Name                                      | Type | Length | Value                            |
| ----------------------------------------- | ---- | ------ | -------------------------------- |
| Operation Type                            | 1    | 1      | 1: List <br>2: Add <br>3: Remove |
| NFC Access Issuer Key Request             | 2    | 1      | TLV8                             |
| NFC Access Issuer Key Response            | 3    | 1      | TLV8                             |
| NFC Access Device Credential Key Request  | 4    | 1      | TLV8                             |
| NFC Access Device Credential Key Response | 5    | 1      | TLV8                             |
| NFC Access Reader Key Request             | 6    | 1      | TLV8                             |
| NFC Access Reader Key Response            | 7    | 1      | TLV8                             |

### NFC Access Issuer Key

`Identifier` field value is calculateed as first 8 bytes of SHA256("key-identifier" || `Key`).

The controller does not call the issuer key addition API during pairing; it is necessary to retrieve and add them during pairing manually

#### Request

| Name       | Type | Length | Value                 | Operation | Description                                                                                     |
| ---------- | ---- | ------ | --------------------- | --------- | ----------------------------------------------------------------------------------------------- |
| Type       | 1    | 1      | [Key Type](#key-type) | Add       | Only Ed25519 is supported for Issuer Key ???                                                    |
| Key        | 2    | TLV8   | Key value             | Add       | Public key used to authenticate the Unified Access Document presented to the accessory over NFC |
| Identifier | 3    | 8      | 8 byte identifier     | Remove    | Unique identifier                                                                               |

#### Response

| Name        | Type | Length | Value                       | Operation         | Description       |
| ----------- | ---- | ------ | --------------------------- | ----------------- | ----------------- |
| Identifier  | 1    | 8      | 8 byte identifier           | List              | Unique identifier |
| Status Code | 2    | 1      | [Status Code](#status-code) | List, Add, Remove |                   |

#### Examples

##### List

Request:

```
01 01 01                    --> Operation Type: List
02 00                       --> NFC Access Issuer Key Request
```

Response:

```
01 01 01                    --> Operation Type: List
03 0D                       --> NFC Access Issuer Key Response
    01 08                   --> Identifier
          AB CD EF 01 23 45 67 89
    02 01 00                --> Status Code: Success
```

##### Add

Request:

```
01 01 02                    --> Operation Type: Add
02 25                       --> NFC Access Issuer Key Request
    01 01 01                --> Type: Ed25519
    02 20                   --> Key
          01 23 45 67 89 AB CD EF 01 23 45 67 89 AB CD EF 01 23 45 67 89 AB CD EF 01 23 45 67 89 AB CD EF
```

Response:

```
01 01 02                    --> Operation Type: Add
03 0D                       --> NFC Access Issuer Key Response
    01 08                   --> Identifier
          FE DC BA 98 76 54 32 10
    02 01 00                --> Status Code: Success
```

##### Remove

Request:

```
01 01 03                    --> Operation Type: Remove
02 0A                       --> NFC Access Issuer Key Request
    03 08                   --> Identifier
          11 22 33 44 55 66 77 88
```

Response:

```
01 01 03                    --> Operation Type: Remove
03 03                       --> NFC Access Issuer Key Response
    02 01 00                --> Status Code: Success
```

### NFC Access Device Credential Key

`Identifier` field value is calculateed as first 8 bytes of SHA256("key-identifier" || `Key`).

Device Credential Keys will not be re-added by the controller if they are lost by the device.

To update the state of an existing Device Credential Key, the controller can perform an add request with the same key and the desired state. In this case, the accessory should properly handle the state change instead of returning a duplicate error.

#### Request

| Name                  | Type | Length | Value                      | Operation | Description                    |
| --------------------- | ---- | ------ | -------------------------- | --------- | ------------------------------ |
| Type                  | 1    | 1      | [Key Type](#key-type)      | Add       | Only NIST256 is supported here |
| Key                   | 2    | TLV8   | Key value                  | Add       | Public key                     |
| Issuer Key Identifier | 3    | TLV8   |                            | Add       |                                |
| State                 | 4    | 1      | 0: Suspended <br>1: Active | List, Add |                                |
| Identifier            | 5    | 8      | 8 byte identifier          | Remove    |                                |

#### Response

| Name                  | Type | Length | Value                       | Operation         |
| --------------------- | ---- | ------ | --------------------------- | ----------------- |
| Identifier            | 1    | 8      | 8 byte identifier           | List              |
| Issuer Key Identifier | 2    | TLV8   |                             | List              |
| Status Code           | 3    | 1      | [Status Code](#status-code) | List, Add, Remove |

#### Examples

##### List

Request:

```
01 01 01                    --> Operation Type: List
04 00                       --> NFC Access Device Credential Key Request
```

Response:

```
01 01 01                    --> Operation Type: List
05 30                       --> NFC Access Device Credential Key Response
    01 08                   --> Identifier
          AA BB CC DD EE FF 00 11
    02 08                   --> Issuer Key Identifier
          22 33 44 55 66 77 88 99
    03 01 00                --> Status Code: Success
00 00                       --> Padding
    01 08                   --> Identifier
          BB CC DD EE FF 00 11 22
    02 08                   --> Issuer Key Identifier
          33 44 55 66 77 88 99 AA
    03 01 00                --> Status Code: Success
```

##### Add

Request:

```
01 01 02                    --> Operation Type: Add
04 52                       --> NFC Access Device Credential Key Request
    01 01 02                --> Type: NIST256
    02 40                   --> Key
          11 22 33 44 55 66 77 88 99 AA BB CC DD EE FF 00 11 22 33 44 55 66 77 88 99 AA BB CC DD EE FF 00 11 22 33 44 55 66 77 88 99 AA BB CC DD EE FF 00 11 22 33 44 55 66 77 88 99 AA BB CC DD EE FF 00
    03 08                   --> Issuer Key Identifier
          22 33 44 55 66 77 88 99
    04 01 01                --> State: Active
```

Response:

```
01 01 02                    --> Operation Type: Add
05 17                       --> NFC Access Device Credential Key Response
    01 08                   --> Identifier
          CC DD EE FF 00 11 22 33
    02 08                   --> Issuer Key Identifier
          22 33 44 55 66 77 88 99
    03 01 00                --> Status Code: Success
```

##### Remove

Request:

```
01 01 03                    --> Operation Type: Remove
04 0A                       --> NFC Access Device Credential Key Request
    05 08                   --> Identifier
          DD EE FF 00 11 22 33 44
```

Response:

```
01 01 03                    --> Operation Type: Remove
05 03                       --> NFC Access Device Credential Key Response
    03 01 00                --> Status Code: Success
```

### NFC Access Reader Key

`Identifier` field value is calculateed as first 8 bytes of SHA256("key-identifier" || `Key`).

There is always one Reader Key per device and it is added by the controller during pairing.

#### Request

| Name              | Type | Length | Value                 | Operation | Description                    |
| ----------------- | ---- | ------ | --------------------- | --------- | ------------------------------ |
| Type              | 1    | 1      | [Key Type](#key-type) | Add       | Only NIST256 is supported here |
| Key               | 2    | TLV8   | Key value             | Add       | Private key                    |
| Reader Identifier | 3    | TLV8   |                       | Add       | Not used                       |
| Identifier        | 4    | TLV8   | 8 byte identifier     | Remove    |                                |

#### Response

| Name        | Type | Length | Value                       | Operation         |
| ----------- | ---- | ------ | --------------------------- | ----------------- |
| Identifier  | 1    | 1      | 8 byte identifier           | List              |
| Status Code | 2    | 1      | [Status Code](#status-code) | List, Add, Remove |

#### Examples

##### List

Request:

```
01 01 01                    --> Operation Type: List
06 00                       --> NFC Access Reader Key Request
```

Response:

```
01 01 01                    --> Operation Type: List
07 0D                       --> NFC Access Reader Key Response
    01 08                   --> Identifier
          EE FF 00 11 22 33 44 55
    02 01 00                --> Status Code: Success
```

##### Add

Request:

```
01 01 02                    --> Operation Type: Add
06 2F                       --> NFC Access Reader Key Request
    01 01 02                --> Type: NIST256
    02 20                   --> Key
          FF EE DD CC BB AA 99 88 77 66 55 44 33 22 11 00 FF EE DD CC BB AA 99 88 77 66 55 44 33 22 11 00
    03 08                   --> Reader Identifier
          EE FF 00 11 22 33 44 55
```

Response:

```
01 01 02                    --> Operation Type: Add
07 0D                       --> NFC Access Reader Key Response
    01 08                   --> Identifier
          EE FF 00 11 22 33 44 55
    02 01 00                --> Status Code: Success
```

##### Remove

Request:

```
01 01 03                    --> Operation Type: Remove
06 0A                       --> NFC Access Reader Key Request
    04 08                   --> Identifier
          EE FF 00 11 22 33 44 55
```

Response:

```
01 01 03                    --> Operation Type: Remove
07 03                       --> NFC Access Reader Key Response
    02 01 00                --> Status Code: Success
```

## NFC Access Supported Configuration

Read-only characteristic that provides information about NFC access capabilities and current configuration.

| Name                                     | Type | Length | Value  |
| ---------------------------------------- | ---- | ------ | ------ |
| Maximum Issuer Keys                      | 1    | TLV8   | Number |
| Maximum Suspended Device Credential Keys | 2    | TLV8   | Number |
| Maximum Active Device Credential Keys    | 3    | TLV8   | Number |

### Examples

Response:

```
01 01 0A                    --> Maximum Issuer Keys: 10
02 01 0A                    --> Maximum Suspended Device Credential Keys: 10
03 01 0A                    --> Maximum Active Device Credential Keys: 10
```
