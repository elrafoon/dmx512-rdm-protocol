<a id="readme-top"></a>

<div align="center">
  <h1 align="center">dmx512-rdm-protocol</h3>
  <h3 align="center">
    DMX512 and Remote Device Management (RDM) protocol written in Rust
  </h3>
  <div align="center">

[![Crates.io](https://img.shields.io/crates/v/dmx512-rdm-protocol.svg)](https://crates.io/crates/dmx512-rdm-protocol)
[![Docs](https://img.shields.io/badge/docs-latest-blue)](https://docs.rs/dmx512-rdm-protocol/latest/dmx512_rdm_protocol/)
[![Docs](https://img.shields.io/badge/msrv-1.81.0-red)](https://docs.rs/dmx512-rdm-protocol/latest/dmx512_rdm_protocol/)

  </div>
</div>

## About the project

DMX512 is a unidirectional packet based communication protocol commonly used to control lighting and effects.

Remote Device Management (RDM) is a backward-compatible extension of DMX512, enabling bi-directional communication with compatible devices for the purpose of discovery, identification, reporting, and configuration.

### Included

- Data-types and functionality for encoding and decoding DMX512 and RDM packets.
- `no_std` implementations by disabling `alloc` feature flag

### Not included

- Driver implementations: DMX512 / RDM packets are transmitted over an RS485 bus but interface devices like the Enttec DMX USB PRO exist to communicate with devices via USB. These interface devices usually require extra packet framing ontop of the standard DMX512 / RDM packets, which is out of scope of this project.

### Implemented specifications / supported parameters

- ANSI E1.11 (2008): Asynchronous Serial Digital Data Transmission Standard for Controlling Lighting Equipment and Accessories [Implemented]
- ANSI E1.20 (2010): RDM Remote Device Management Over DMX512 Networks [Implemented]
- ANSI E1.37-1 (2012): Additional Message Sets for ANSI E1.20 (RDM) – Part 1, Dimmer Message Sets [Implemented]
- ANSI E1.37-2 (2015): Additional Message Sets for ANSI E1.20 (RDM) – Part 2, IPv4 & DNS Configuration Messages [Implemented]
- ANSI E1.37-7 (2019): Additional Message Sets for ANSI E1.20 (RDM) – Gateway & Splitter Configuration Messages [Implemented]
- ANSI E1.33 (RDMnet): Message Transport and Management for ANSI E1.20 (RDM) compatible and similar devices over IP Networks [Supported]

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Installation

```sh
cargo add dmx512-rdm-protocol
```

or add to Cargo.toml dependencies, [crates.io](https://crates.io/crates/dmx512-rdm-protocol) for latest version.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Feature flags

The default-features for this project are heap allocation implementations and rdm features included, however if you just want to use the basic DMX512 data-types and functionality and it be `no_std` compatible, you can set `default-features = false` in the Cargo.toml dependency.

Otherwise the following flags can be toggled to find subset of the functionality.

- Add `rdm` flag to conditionally compile rdm features. The `rdm` features have `no_std` compatible implementations.
- Add `alloc` flag for heap allocation implementation, i.e not `no_std` compatible.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Usage

### DmxUniverse

```rust
use dmx512_rdm_protocol::dmx::DmxUniverse;

// Create a 512 channel universe
let dmx_universe = DmxUniverse::default();
// or create a smaller universe

// If you use the heap allocation implementations, you can create smaller universes
#[cfg(feature = "alloc")]
let dmx_universe = DmxUniverse::new(4).unwrap();

// or decode a dmx packet
let mut dmx_universe = DmxUniverse::decode(&[0, 255, 255, 0, 0]).unwrap();

assert_eq!(&dmx_universe.as_slice()[..4], &[255, 255, 0, 0]);

dmx_universe.set_channel_value(0, 64).unwrap();
dmx_universe.set_channel_values(1, &[128, 192, 255]).unwrap();

assert_eq!(dmx_universe.get_channel_value(0).unwrap(), 64);
assert_eq!(dmx_universe.get_channel_values(1..=2).unwrap(), &[128, 192]);
assert_eq!(&dmx_universe.as_slice()[..4], &[64, 128, 192, 255]);
assert_eq!(&dmx_universe.encode()[..5], &[0, 64, 128, 192, 255]);
```

### RdmRequest

```rust
let encoded = RdmRequest::new(
    DeviceUID::new(0x0102, 0x03040506),
    DeviceUID::new(0x0605, 0x04030201),
    0x00,
    0x01,
    SubDeviceId::RootDevice,
    RequestParameter::GetIdentifyDevice,
)
.encode();

let expected = &[
    0xcc, // Start Code
    0x01, // Sub Start Code
    0x18, // Message Length
    0x01, 0x02, 0x03, 0x04, 0x05, 0x06, // Destination UID
    0x06, 0x05, 0x04, 0x03, 0x02, 0x01, // Source UID
    0x00, // Transaction Number
    0x01, // Port ID
    0x00, // Message Count
    0x00, 0x00, // Sub-Device ID = Root Device
    0x20, // Command Class = GetCommand
    0x10, 0x00, // Parameter ID = Identify Device
    0x00, // PDL
    0x01, 0x40, // Checksum
];

assert_eq!(encoded, expected);
```

### RdmResponse

```rust
let decoded = RdmResponse::decode(&[
    0xcc, // Start Code
    0x01, // Sub Start Code
    0x19, // Message Length
    0x01, 0x02, 0x03, 0x04, 0x05, 0x06, // Destination UID
    0x06, 0x05, 0x04, 0x03, 0x02, 0x01, // Source UID
    0x00, // Transaction Number
    0x00, // Response Type = Ack
    0x00, // Message Count
    0x00, 0x00, // Sub-Device ID = Root Device
    0x21, // Command Class = GetCommandResponse
    0x10, 0x00, // Parameter ID = Identify Device
    0x01, // PDL
    0x01, // Identifying = true
    0x01, 0x43, // Checksum
]);

let expected = Ok(RdmResponse::RdmFrame(RdmFrameResponse {
    destination_uid: DeviceUID::new(0x0102, 0x03040506),
    source_uid: DeviceUID::new(0x0605, 0x04030201),
    transaction_number: 0x00,
    response_type: ResponseType::Ack,
    message_count: 0x00,
    sub_device_id: SubDeviceId::RootDevice,
    command_class: CommandClass::GetCommandResponse,
    parameter_id: ParameterId::IdentifyDevice,
    parameter_data: ResponseData::ParameterData(Some(
        ResponseParameterData::GetIdentifyDevice(true),
    )),
}));

assert_eq!(decoded, expected);
```

See tests for more examples.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Contributing

This project is open to contributions, create a new issue and let's discuss.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## License

Distributed under the MIT License. See `LICENSE.txt` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Acknowledgments

- The ANSI E1.11 (2008), ANSI E1.20 (2010), ANSI E1.37-1 (2012), ANSI E1.37-2 (2015), ANSI E1.37-7 (2019) and ANSI E1.33 (RDMnet) specifications used to create this library is copyright and published by [ESTA](https://www.esta.org/)

<p align="right">(<a href="#readme-top">back to top</a>)</p>
