# UART Register Verification Plan (VPlan)
## ARM PL011 UART — UVM Register Abstraction Layer (RAL)

---

**Document Control**

| Attribute      | Value                                        |
|----------------|----------------------------------------------|
| Document Title | UART Register Verification Plan              |
| Component      | ARM PL011 UART (PrimeCell)                   |
| Version        | r1p5_1                                       |
| Vendor         | arm.com                                      |
| Source         | ARM_UART.regman (rtl_1p0_release_regman)     |
| Date           | 2026-06-26                                   |
| Status         | Draft                                        |

---

## Table of Contents

1. Document Overview
2. DUT Register Overview
3. Verification Objectives
4. Verification Strategy
5. Verification Features
6. Test Plan
7. Functional Coverage Plan
8. Requirements Traceability Matrix (RTM)
9. Regression Plan
10. Exit Criteria

---

## 1. Document Overview

### 1.1 Purpose

This Verification Plan (VPlan) defines the register-level verification strategy and test plan for the ARM PL011 UART peripheral. It covers all registers defined in the `ARM_UART.regman` file using the UVM Register Abstraction Layer (RAL). The plan specifies frontdoor and backdoor access strategies, reset verification, access-policy verification, field-level checks, reserved-bit handling, and negative testing for every register and field extracted from the RegMan specification.

### 1.2 Scope

This document covers:
- All 26 registers in the `APB_Slave_MM` address map of the ARM PL011 UART
- All named bitfields within those registers
- Access-type enforcement for RO, RW, WO fields
- Reset value verification for all fields
- Frontdoor (APB bus) and backdoor (HDL path) verification
- Functional coverage closure at register, field, and access-type granularity

This document does **not** cover:
- Protocol-level UART TX/RX data path testing (out of scope for register verification)
- DMA controller internal operation
- IrDA (SIR) serial protocol testing
- Interrupt propagation to the interrupt controller (system-level)

### 1.3 References

| Reference | Description |
|-----------|-------------|
| ARM_UART.regman | ARM PL011 UART Register Map Source — rtl_1p0_release_regman |
| UVM 1.2 Standard | UVM Register Abstraction Layer (uvm_reg) specification |
| AMBA APB3 Specification | ARM AMBA 3 APB Protocol Specification |

---

## 2. DUT Register Overview

### 2.1 Component Summary

| Parameter        | Value                                           |
|------------------|-------------------------------------------------|
| Component Name   | Uart                                            |
| Full Name        | ARM PrimeCell UART (PL011)                      |
| Version          | r1p5_1                                          |
| Library          | PrimeCell                                       |
| Vendor           | arm.com                                         |
| Role             | Peripheral                                      |
| Bus Interface    | APB3 (APB Slave — APB_Slave)                    |
| Address Width    | 32-bit                                          |
| Data Width       | 32-bit                                          |
| Address Map Size | 0x1000 (4 KB)                                   |
| Address Unit     | 8-bit (byte addressing)                         |
| Error Response   | Supported                                       |

### 2.2 Register Map

| #  | Register Name       | Offset  | Width  | Reg Access | Description                                  |
|----|---------------------|---------|--------|------------|----------------------------------------------|
| 1  | UARTDR              | 0x000   | 16-bit | RW         | UART Data Register                           |
| 2  | UARTRSR_UARTECR     | 0x004   | 8-bit  | RW         | Receive Status / Error Clear Register        |
| 3  | UARTFR              | 0x018   | 16-bit | RO         | Flag Register                                |
| 4  | UARTILPR            | 0x020   | 8-bit  | RW         | IrDA Low-Power Counter Register              |
| 5  | UARTIBRD            | 0x024   | 16-bit | RW         | Integer Baud Rate Register                   |
| 6  | UARTFBRD            | 0x028   | 8-bit  | RW         | Fractional Baud Rate Register                |
| 7  | UARTLCR_H           | 0x02C   | 16-bit | RW         | Line Control Register                        |
| 8  | UARTCR              | 0x030   | 16-bit | RW         | Control Register                             |
| 9  | UARTIFLS            | 0x034   | 16-bit | RW         | Interrupt FIFO Level Select Register         |
| 10 | UARTIMSC            | 0x038   | 16-bit | RW         | Interrupt Mask Set/Clear Register            |
| 11 | UARTRIS             | 0x03C   | 16-bit | RO         | Raw Interrupt Status Register                |
| 12 | UARTMIS             | 0x040   | 16-bit | RO         | Masked Interrupt Status Register             |
| 13 | UARTICR             | 0x044   | 16-bit | RW         | Interrupt Clear Register                     |
| 14 | UARTDMACR           | 0x048   | 16-bit | RW         | DMA Control Register                         |
| 15 | UARTTCR             | 0x080   | 16-bit | RW         | Test Control Register                        |
| 16 | UARTITIP            | 0x084   | 16-bit | RW         | Integration Test Input Register              |
| 17 | UARTITOP            | 0x088   | 16-bit | RW         | Integration Test Output Register             |
| 18 | UARTTDR             | 0x08C   | 16-bit | RW         | Test Data Register                           |
| 19 | UARTPeriphID0       | 0xFE0   | 16-bit | RO         | Peripheral Identification Register 0        |
| 20 | UARTPeriphID1       | 0xFE4   | 16-bit | RO         | Peripheral Identification Register 1        |
| 21 | UARTPeriphID2       | 0xFE8   | 16-bit | RO         | Peripheral Identification Register 2        |
| 22 | UARTPeriphID3       | 0xFEC   | 16-bit | RO         | Peripheral Identification Register 3        |
| 23 | UARTPCellID0        | 0xFF0   | 16-bit | RO         | PrimeCell Identification Register 0         |
| 24 | UARTPCellID1        | 0xFF4   | 16-bit | RO         | PrimeCell Identification Register 1         |
| 25 | UARTPCellID2        | 0xFF8   | 16-bit | RO         | PrimeCell Identification Register 2         |
| 26 | UARTPCellID3        | 0xFFC   | 16-bit | RO         | PrimeCell Identification Register 3         |

### 2.3 Field Summary

#### UARTDR (0x000) — Data Register

| Field     | Bits  | Width | SW Access | HW Access | Reset | Description                          |
|-----------|-------|-------|-----------|-----------|-------|--------------------------------------|
| Reserved  | 15:12 | 4     | RO        | RO        | 0x0   | Reserved                             |
| OE        | 11    | 1     | RO        | RO        | 0     | Overrun error                        |
| BE        | 10    | 1     | RO        | RO        | 0     | Break error                          |
| PE        | 9     | 1     | RO        | RO        | 0     | Parity error                         |
| FE        | 8     | 1     | RO        | RO        | 0     | Framing error                        |
| DATA      | 7:0   | 8     | RW        | RW        | 0x00  | Receive (read) / Transmit (write)    |

#### UARTRSR_UARTECR (0x004) — Receive Status / Error Clear Register

| Field        | Bits | Width | SW Access | HW Access | Reset | Description                              |
|--------------|------|-------|-----------|-----------|-------|------------------------------------------|
| Reserved     | 7:4  | 4     | RO        | RO        | 0x0   | Reserved, unpredictable when read        |
| OE           | 3    | 1     | RO        | RO        | 0     | Overrun error status                     |
| BE           | 2    | 1     | RO        | RO        | 0     | Break error status                       |
| PE           | 1    | 1     | RO        | RO        | 0     | Parity error status                      |
| FE           | 0    | 1     | RO        | RO        | 0     | Framing error status                     |
| Clear_Errors | 7:0  | 8     | WO        | Default   | 0x00  | Write any value to clear all error flags |

*Note: This register has a dual-access personality. A read returns the 4-bit error status (OE/BE/PE/FE) in bits [3:0]. A write to any data value clears all four error flags. The write-only field Clear_Errors overlaps with the read-only error fields in the physical address.*

#### UARTFR (0x018) — Flag Register

| Field    | Bits | Width | SW Access | HW Access | Reset | Description                            |
|----------|------|-------|-----------|-----------|-------|----------------------------------------|
| Reserved | 15:9 | 7     | RO        | RO        | 0x0   | Reserved, read as zero                 |
| RI       | 8    | 1     | RO        | RO        | 0     | Ring indicator (complement of nUARTRI) |
| TXFE     | 7    | 1     | RO        | RO        | 1     | TX FIFO empty                          |
| RXFF     | 6    | 1     | RO        | RO        | 0     | RX FIFO full                           |
| TXFF     | 5    | 1     | RO        | RO        | 0     | TX FIFO full                           |
| RXFE     | 4    | 1     | RO        | RO        | 1     | RX FIFO empty                          |
| BUSY     | 3    | 1     | RO        | RO        | 0     | UART busy transmitting                 |
| DCD      | 2    | 1     | RO        | RO        | 0     | Data carrier detect                    |
| DSR      | 1    | 1     | RO        | RO        | 0     | Data set ready                         |
| CTS      | 0    | 1     | RO        | RO        | 0     | Clear to send                          |

#### UARTILPR (0x020) — IrDA Low-Power Counter Register

| Field    | Bits | Width | SW Access | HW Access | Reset | Description                   |
|----------|------|-------|-----------|-----------|-------|-------------------------------|
| ILPDVSR  | 7:0  | 8     | RW        | Default   | 0x00  | 8-bit low-power divisor value |

#### UARTIBRD (0x024) — Integer Baud Rate Register

| Field       | Bits  | Width | SW Access | HW Access | Reset  | Description                  |
|-------------|-------|-------|-----------|-----------|--------|------------------------------|
| BAUD_DIVINT | 15:0  | 16    | RW        | Default   | 0x0000 | Integer baud rate divisor    |

#### UARTFBRD (0x028) — Fractional Baud Rate Register

| Field        | Bits | Width | SW Access | HW Access | Reset | Description                    |
|--------------|------|-------|-----------|-----------|-------|--------------------------------|
| BAUD_DIVFRAC | 5:0  | 6     | RW        | Default   | 0x00  | Fractional baud rate divisor   |

*Note: Bits [7:6] of this 8-bit register are not defined as named fields in the RegMan. Behavior of bits [7:6] is not specified in the source.*

#### UARTLCR_H (0x02C) — Line Control Register

| Field    | Bits | Width | SW Access | HW Access | Reset | Description                                              |
|----------|------|-------|-----------|-----------|-------|----------------------------------------------------------|
| Reserved | 15:8 | 8     | RO        | RO        | 0x0   | Reserved, read as zero                                   |
| SPS      | 7    | 1     | RW        | Default   | 0     | Stick parity select                                      |
| WLEN     | 6:5  | 2     | RW        | Default   | 0b00  | Word length (00=5b, 01=6b, 10=7b, 11=8b)                |
| FEN      | 4    | 1     | RW        | Default   | 0     | Enable FIFOs (0=character mode, 1=FIFO mode)             |
| STP2     | 3    | 1     | RW        | Default   | 0     | Two stop bits select                                     |
| EPS      | 2    | 1     | RW        | Default   | 0     | Even parity select (0=odd, 1=even)                       |
| PEN      | 1    | 1     | RW        | Default   | 0     | Parity enable                                            |
| BRK      | 0    | 1     | RW        | Default   | 0     | Send break (0=normal, 1=send break)                      |

#### UARTCR (0x030) — Control Register

| Field    | Bits  | Width | SW Access | HW Access | Reset | Description                                      |
|----------|-------|-------|-----------|-----------|-------|--------------------------------------------------|
| CTSEn    | 15    | 1     | RW        | Default   | 0     | CTS hardware flow control enable                 |
| RTSEn    | 14    | 1     | RW        | Default   | 0     | RTS hardware flow control enable                 |
| Out2     | 13    | 1     | RW        | Default   | 0     | Complement of nUARTOut2 (modem output)           |
| Out1     | 12    | 1     | RW        | Default   | 0     | Complement of nUARTOut1 (modem output)           |
| RTS      | 11    | 1     | RW        | Default   | 0     | Request to send (complement of nUARTRTS)         |
| DTR      | 10    | 1     | RW        | Default   | 0     | Data transmit ready (complement of nUARTDTR)     |
| RXE      | 9     | 1     | RW        | Default   | 1     | Receive enable                                   |
| TXE      | 8     | 1     | RW        | Default   | 1     | Transmit enable                                  |
| LBE      | 7     | 1     | RW        | Default   | 0     | Loopback enable                                  |
| Reserved | 6:3   | 4     | RO        | RO        | 0x0   | Reserved, read as zero                           |
| SIRLP    | 2     | 1     | RW        | Default   | 0     | SIR low-power IrDA mode                          |
| SIREN    | 1     | 1     | RW        | Default   | 0     | SIR enable (IrDA SIR ENDEC)                      |
| UARTEN   | 0     | 1     | RW        | Default   | 0     | UART enable                                      |

#### UARTIFLS (0x034) — Interrupt FIFO Level Select Register

| Field    | Bits | Width | SW Access | HW Access | Reset | Description                                                    |
|----------|------|-------|-----------|-----------|-------|----------------------------------------------------------------|
| Reserved | 15:6 | 10    | RO        | RO        | 0x0   | Reserved, read as zero                                         |
| RXIFLSEL | 5:3  | 3     | RW        | Default   | 0b010 | RX interrupt FIFO level (000=1/8, 001=1/4, 010=1/2, 011=3/4, 100=7/8) |
| TXIFLSEL | 2:0  | 3     | RW        | Default   | 0b010 | TX interrupt FIFO level (000=1/8, 001=1/4, 010=1/2, 011=3/4, 100=7/8) |

#### UARTIMSC (0x038) — Interrupt Mask Set/Clear Register

| Field   | Bits  | Width | SW Access | HW Access | Reset | Description                              |
|---------|-------|-------|-----------|-----------|-------|------------------------------------------|
| Reserved| 15:11 | 5     | RO        | RO        | 0x0   | Reserved, read as zero                   |
| OEIM    | 10    | 1     | RW        | Default   | 0     | Overrun error interrupt mask             |
| BEIM    | 9     | 1     | RW        | Default   | 0     | Break error interrupt mask               |
| PEIM    | 8     | 1     | RW        | Default   | 0     | Parity error interrupt mask              |
| FEIM    | 7     | 1     | RW        | Default   | 0     | Framing error interrupt mask             |
| RTIM    | 6     | 1     | RW        | Default   | 0     | Receive timeout interrupt mask           |
| TXIM    | 5     | 1     | RW        | Default   | 0     | Transmit interrupt mask                  |
| RXIM    | 4     | 1     | RW        | Default   | 0     | Receive interrupt mask                   |
| DSRMIM  | 3     | 1     | RW        | Default   | 0     | DSR modem interrupt mask                 |
| DCDMIM  | 2     | 1     | RW        | Default   | 0     | DCD modem interrupt mask                 |
| CTSMIM  | 1     | 1     | RW        | Default   | 0     | CTS modem interrupt mask                 |
| RIMIM   | 0     | 1     | RW        | Default   | 0     | RI modem interrupt mask                  |

#### UARTRIS (0x03C) — Raw Interrupt Status Register

| Field   | Bits  | Width | SW Access | HW Access | Reset | Description                              |
|---------|-------|-------|-----------|-----------|-------|------------------------------------------|
| Reserved| 15:11 | 5     | RO        | RO        | 0x0   | Reserved, read as zero                   |
| OERIS   | 10    | 1     | RO        | RO        | 0     | Overrun error raw interrupt status       |
| BERIS   | 9     | 1     | RO        | RO        | 0     | Break error raw interrupt status         |
| PERIS   | 8     | 1     | RO        | RO        | 0     | Parity error raw interrupt status        |
| FERIS   | 7     | 1     | RO        | RO        | 0     | Framing error raw interrupt status       |
| RTRIS   | 6     | 1     | RO        | RO        | 0     | Receive timeout raw interrupt status     |
| TXRIS   | 5     | 1     | RO        | RO        | 0     | Transmit raw interrupt status            |
| RXRIS   | 4     | 1     | RO        | RO        | 0     | Receive raw interrupt status             |
| DSRRMIS | 3     | 1     | RO        | RO        | 0     | DSR modem raw interrupt status           |
| DCDRMIS | 2     | 1     | RO        | RO        | 0     | DCD modem raw interrupt status           |
| CTSRMIS | 1     | 1     | RO        | RO        | 0     | CTS modem raw interrupt status           |
| RIRMIS  | 0     | 1     | RO        | RO        | 0     | RI modem raw interrupt status            |

#### UARTMIS (0x040) — Masked Interrupt Status Register

| Field   | Bits  | Width | SW Access | HW Access | Reset | Description                               |
|---------|-------|-------|-----------|-----------|-------|-------------------------------------------|
| Reserved| 15:11 | 5     | RO        | RO        | 0x0   | Reserved, read as zero                    |
| OEMIS   | 10    | 1     | RO        | RO        | 0     | Overrun error masked interrupt status     |
| BEMIS   | 9     | 1     | RO        | RO        | 0     | Break error masked interrupt status       |
| PEMIS   | 8     | 1     | RO        | RO        | 0     | Parity error masked interrupt status      |
| FEMIS   | 7     | 1     | RO        | RO        | 0     | Framing error masked interrupt status     |
| RTMIS   | 6     | 1     | RO        | RO        | 0     | Receive timeout masked interrupt status   |
| TXMIS   | 5     | 1     | RO        | RO        | 0     | Transmit masked interrupt status          |
| RXMIS   | 4     | 1     | RO        | RO        | 0     | Receive masked interrupt status           |
| DSRMMIS | 3     | 1     | RO        | RO        | 0     | DSR modem masked interrupt status         |
| DCDMMIS | 2     | 1     | RO        | RO        | 0     | DCD modem masked interrupt status         |
| CTSMMIS | 1     | 1     | RO        | RO        | 0     | CTS modem masked interrupt status         |
| RIMMIS  | 0     | 1     | RO        | RO        | 0     | RI modem masked interrupt status          |

#### UARTICR (0x044) — Interrupt Clear Register

| Field   | Bits  | Width | SW Access | HW Access | Reset | Description                            |
|---------|-------|-------|-----------|-----------|-------|----------------------------------------|
| Reserved| 15:11 | 5     | RO        | RO        | 0x0   | Reserved, read as zero                 |
| OEIC    | 10    | 1     | WO        | Default   | 0     | Overrun error interrupt clear          |
| BEIC    | 9     | 1     | WO        | Default   | 0     | Break error interrupt clear            |
| PEIC    | 8     | 1     | WO        | Default   | 0     | Parity error interrupt clear           |
| FEIC    | 7     | 1     | WO        | Default   | 0     | Framing error interrupt clear          |
| RTIC    | 6     | 1     | WO        | Default   | 0     | Receive timeout interrupt clear        |
| TXIC    | 5     | 1     | WO        | Default   | 0     | Transmit interrupt clear               |
| RXIC    | 4     | 1     | WO        | Default   | 0     | Receive interrupt clear                |
| DSRMIC  | 3     | 1     | WO        | Default   | 0     | DSR modem interrupt clear              |
| DCDMIC  | 2     | 1     | WO        | Default   | 0     | DCD modem interrupt clear              |
| CTSMIC  | 1     | 1     | WO        | Default   | 0     | CTS modem interrupt clear              |
| RIMIC   | 0     | 1     | WO        | Default   | 0     | RI modem interrupt clear               |

#### UARTDMACR (0x048) — DMA Control Register

| Field    | Bits  | Width | SW Access | HW Access | Reset | Description                                       |
|----------|-------|-------|-----------|-----------|-------|---------------------------------------------------|
| Reserved | 15:3  | 13    | RO        | RO        | 0x0   | Reserved, read as zero                            |
| DMAONERR | 2     | 1     | RW        | Default   | 0     | DMA on error (disable DMA RX request on error)   |
| TXDMAE   | 1     | 1     | RW        | Default   | 0     | Transmit DMA enable                               |
| RXDMAE   | 0     | 1     | RW        | Default   | 0     | Receive DMA enable                                |

#### UARTTCR (0x080) — Test Control Register

| Field    | Bits  | Width | SW Access | HW Access | Reset | Description                                           |
|----------|-------|-------|-----------|-----------|-------|-------------------------------------------------------|
| Reserved | 15:3  | 13    | RO        | RO        | 0x0   | Reserved, unpredictable when read                     |
| SIRTEST  | 2     | 1     | RW        | Default   | 0     | SIR test enable (full-duplex SIR for test)            |
| TESTFIFO | 1     | 1     | RW        | Default   | 0     | Test FIFO enable (write to RX FIFO / read TX FIFO)   |
| ITEN     | 0     | 1     | RW        | Default   | 0     | Integration test enable                               |

#### UARTITIP (0x084) — Integration Test Input Register

| Field        | Bits | Width | SW Access | HW Access | Reset | Description                         |
|--------------|------|-------|-----------|-----------|-------|-------------------------------------|
| Reserved     | 15:8 | 8     | RO        | RO        | 0x0   | Reserved, unpredictable when read   |
| UARTTXDMACLR | 7    | 1     | RW        | RW        | 0     | TX DMA clear (integration test)     |
| UARTRXDMACLR | 6    | 1     | RW        | RW        | 0     | RX DMA clear (integration test)     |
| nUARTRI      | 5    | 1     | RW        | RW        | 0     | Ring indicator primary input        |
| nUARTDCD     | 4    | 1     | RW        | RW        | 0     | DCD primary input                   |
| nUARTCTS     | 3    | 1     | RW        | RW        | 0     | CTS primary input                   |
| nUARTDSR     | 2    | 1     | RW        | RW        | 0     | DSR primary input                   |
| SIRIN        | 1    | 1     | RW        | RW        | 0     | SIR input                           |
| UARTRXD      | 0    | 1     | RW        | RW        | 0     | UART RX data input                  |

#### UARTITOP (0x088) — Integration Test Output Register

| Field         | Bits | Width | SW Access | HW Access | Reset | Description                              |
|---------------|------|-------|-----------|-----------|-------|------------------------------------------|
| UARTTXDMASREQ | 15   | 1     | RW        | RW        | 0     | TX DMA single request (intra-chip)       |
| UARTTXDMABREQ | 14   | 1     | RW        | RW        | 0     | TX DMA burst request (intra-chip)        |
| UARTRXDMASREQ | 13   | 1     | RW        | RW        | 0     | RX DMA single request (intra-chip)       |
| UARTRXDMABREQ | 12   | 1     | RW        | RW        | 0     | RX DMA burst request (intra-chip)        |
| UARTMSINTR    | 11   | 1     | RW        | RW        | 0     | Modem status interrupt (intra-chip)      |
| UARTRXINTR    | 10   | 1     | RW        | RW        | 0     | RX interrupt (intra-chip)                |
| UARTTXINTR    | 9    | 1     | RW        | RW        | 0     | TX interrupt (intra-chip)                |
| UARTRTINTR    | 8    | 1     | RW        | RW        | 0     | Receive timeout interrupt (intra-chip)   |
| UARTEINTR     | 7    | 1     | RW        | RW        | 0     | Error interrupt (intra-chip)             |
| UARTINTR      | 6    | 1     | RW        | RW        | 0     | Combined interrupt (intra-chip)          |
| nUARTOut2     | 5    | 1     | RW        | RW        | 0     | Out2 primary output                      |
| nUARTOut1     | 4    | 1     | RW        | RW        | 0     | Out1 primary output                      |
| nUARTRTS      | 3    | 1     | RW        | RW        | 0     | RTS primary output                       |
| nUARTDTR      | 2    | 1     | RW        | RW        | 0     | DTR primary output                       |
| nSIROUT       | 1    | 1     | RW        | RW        | 0     | SIR output primary                       |
| UARTTXD       | 0    | 1     | RW        | RW        | 0     | UART TX data output                      |

#### UARTTDR (0x08C) — Test Data Register

| Field    | Bits  | Width | SW Access | HW Access | Reset | Description                                       |
|----------|-------|-------|-----------|-----------|-------|---------------------------------------------------|
| Reserved | 15:11 | 5     | RO        | RO        | 0x0   | Reserved, unpredictable when read                 |
| DATA     | 10:0  | 11    | RW        | RW        | 0x000 | Test data (write → RX FIFO, read → TX FIFO)      |

#### UARTPeriphID0–3 (0xFE0–0xFEC) — Peripheral Identification Registers

| Register       | Offset | Field        | Bits | Reset  | Description                         |
|----------------|--------|--------------|------|--------|-------------------------------------|
| UARTPeriphID0  | 0xFE0  | PartNumber0  | 7:0  | 0x11   | Part number LSB                     |
| UARTPeriphID1  | 0xFE4  | Designer0    | 7:4  | 0x1    | Designer code MSB                   |
| UARTPeriphID1  | 0xFE4  | PartNumber1  | 3:0  | 0x0    | Part number MSB                     |
| UARTPeriphID2  | 0xFE8  | Revision     | 7:4  | 0x3    | Revision (r1p4=0x2, r1p5=0x3)      |
| UARTPeriphID2  | 0xFE8  | Designer1    | 3:0  | 0x4    | Designer code LSB                   |
| UARTPeriphID3  | 0xFEC  | Configuration| 7:0  | 0x00   | Configuration                       |

#### UARTPCellID0–3 (0xFF0–0xFFC) — PrimeCell Identification Registers

| Register       | Offset | Field        | Bits | Reset | Description             |
|----------------|--------|--------------|------|-------|-------------------------|
| UARTPCellID0   | 0xFF0  | UARTPCellID0 | 7:0  | 0x0D  | PrimeCell ID byte 0     |
| UARTPCellID1   | 0xFF4  | UARTPCellID1 | 7:0  | 0xF0  | PrimeCell ID byte 1     |
| UARTPCellID2   | 0xFF8  | UARTPCellID2 | 7:0  | 0x05  | PrimeCell ID byte 2     |
| UARTPCellID3   | 0xFFC  | UARTPCellID3 | 7:0  | 0xB1  | PrimeCell ID byte 3     |

---

## 3. Verification Objectives

| Obj# | Objective                                                                                        |
|------|--------------------------------------------------------------------------------------------------|
| OBJ-1 | Verify that every register resets to its specified value after a hardware reset                 |
| OBJ-2 | Verify that all RW fields can be written and read back correctly via frontdoor (APB)            |
| OBJ-3 | Verify that all RO fields cannot be written through frontdoor access (write-protected)          |
| OBJ-4 | Verify that all WO fields can be written but do not return written data on read                  |
| OBJ-5 | Verify that reserved bits read as defined (zero or unpredictable as specified)                  |
| OBJ-6 | Verify the UARTRSR_UARTECR dual-access personality (read returns status, write clears errors)  |
| OBJ-7 | Verify that interrupt mask fields in UARTIMSC correctly gate interrupt status into UARTMIS      |
| OBJ-8 | Verify that UARTICR WO fields correctly clear the corresponding bits in UARTRIS/UARTMIS         |
| OBJ-9 | Verify that UARTIFLS RXIFLSEL and TXIFLSEL reset to 0b010 (1/2 full)                           |
| OBJ-10| Verify UARTCR RXE and TXE reset to 1 (enabled by default)                                     |
| OBJ-11| Verify UARTFR TXFE and RXFE reset to 1 (FIFOs empty at reset)                                 |
| OBJ-12| Verify all Peripheral ID and PrimeCell ID registers return their specified constant values       |
| OBJ-13| Verify backdoor peek/poke access for all writable registers                                    |
| OBJ-14| Verify mirror/update register model operations stay consistent with DUT state                   |
| OBJ-15| Verify WLEN field encodes 5/6/7/8 data bits correctly (all four enumerated values)             |

---

## 4. Verification Strategy

### 4.1 Frontdoor Verification Strategy

Frontdoor access exercises the complete APB3 bus path to and from the UART register file. All frontdoor tests use UVM RAL `write()` and `read()` tasks which translate to APB PWRITE/PREAD transactions.

- **Write path**: RAL model generates APB write transaction. DUT register field is updated. Mirror is updated.
- **Read path**: RAL model generates APB read transaction. Returned value is compared against the predicted (mirrored) value or the expected constant for RO registers.
- The APB adapter must correctly translate `uvm_reg_bus_op` to APB signals (PADDR, PWDATA, PWRITE, PENABLE, PSEL, PRDATA, PREADY).
- All frontdoor writes to RO fields must be confirmed to have no effect.
- All frontdoor writes to WO fields must cause the correct side-effect without readable reflection.

### 4.2 Backdoor Verification Strategy

Backdoor access uses UVM RAL `peek()` and `poke()` tasks, which directly read/write DUT register state via HDL simulation paths without generating APB transactions.

- Backdoor is used to set up DUT state (e.g., set error flags) without exercising the bus.
- Backdoor reads (peek) validate that the internal DUT state matches the RAL mirror.
- Backdoor pokes are followed by frontdoor reads to cross-check consistency between HDL state and APB read.
- Backdoor writes to RO fields (if permitted by the RAL model) must be verified to update DUT state directly.

### 4.3 Register Reset Verification

- Apply hardware reset (PRESETn) and immediately read all registers via frontdoor.
- Compare each field against its `reset` value as specified in the RegMan.
- Special cases:
  - UARTCR: RXE=1, TXE=1 must be asserted at reset (non-zero reset defaults).
  - UARTFR: TXFE=1, RXFE=1 must be asserted at reset.
  - UARTIFLS: RXIFLSEL=0b010, TXIFLSEL=0b010 at reset.
  - UARTPeriphID0: PartNumber0=0x11.
  - UARTPCellID0–3: 0x0D, 0xF0, 0x05, 0xB1.
- Use the UVM RAL built-in `reset()` task to initialize the mirror, then call `read_reg_by_name()` on all registers.

### 4.4 Register Access Policy Verification

The access policy for each field must be verified explicitly:

| Access Type | Field Examples                         | Verification Method                                           |
|-------------|----------------------------------------|---------------------------------------------------------------|
| RW          | UARTIBRD.BAUD_DIVINT, UARTLCR_H.WLEN  | Write known pattern, read back, compare                      |
| RO          | UARTFR.TXFE, UARTDR.OE                 | Write 1s, verify no change on readback                       |
| WO          | UARTICR.OEIC, UARTRSR_UARTECR.Clear_Errors | Write to trigger effect, verify side-effect via RIS/RSR read |

For RW fields: write 0xAA, read back; write 0x55, read back; write 0x00, read back.
For RO fields: write all-1s, read back, compare to expected (reset value or hardware-driven value).
For WO fields: write the field, then read an associated status register to confirm the effect.

### 4.5 Field-Level Verification

Each writable field must be exercised at its legal boundary values:

- **1-bit fields**: Write 0, read; write 1, read.
- **2-bit fields (WLEN)**: Walk through all four states (0b00, 0b01, 0b10, 0b11).
- **3-bit fields (RXIFLSEL, TXIFLSEL)**: Walk through legal states 0–4 (0b101–0b111 are reserved per spec).
- **6-bit field (BAUD_DIVFRAC)**: Write 0x00, 0x3F, 0x15, 0x2A.
- **8-bit fields (ILPDVSR, UARTDR.DATA)**: Write 0x00, 0xFF, 0xA5, 0x5A.
- **11-bit field (UARTTDR.DATA)**: Write 0x000, 0x7FF, 0x555, 0x2AA.
- **16-bit fields (BAUD_DIVINT)**: Write 0x0000, 0xFFFF, 0xAAAA, 0x5555.

### 4.6 Reserved Bit Verification

Reserved fields are categorized based on their specified read behavior:

| Behavior                   | Registers                                            | Verification Method                              |
|----------------------------|------------------------------------------------------|--------------------------------------------------|
| Read as zero               | UARTFR[15:9], UARTLCR_H[15:8], UARTCR[6:3], UARTIFLS[15:6], UARTIMSC[15:11], UARTRIS[15:11], UARTMIS[15:11], UARTICR[15:11], UARTDMACR[15:3] | Read and verify bits are 0                       |
| Unpredictable when read    | UARTRSR_UARTECR[7:4], UARTTCR[15:3], UARTITIP[15:8], UARTTDR[15:11] | Do not check value; verify no exception/error    |
| Read undefined (zeros)     | UARTPeriphIDx[15:8], UARTPCellIDx[15:8]              | Read and verify reserved section reads as zeros  |

Do not write to reserved fields in normal operation. Confirm that reserved fields with "read as zero" behavior always return 0 regardless of other operations on the register.

### 4.7 Negative Testing Strategy

| Scenario                                          | Description                                               |
|---------------------------------------------------|-----------------------------------------------------------|
| Write to RO register (UARTFR, UARTRIS, UARTMIS)  | Attempt write, verify no exception and no field change    |
| Write to RO field within RW register              | Write to UARTDR[11:8] error flags, verify no change       |
| Write reserved value to RXIFLSEL/TXIFLSEL         | Write 0b101–0b111, verify no functional corruption        |
| Read from WO field (UARTICR interrupt clears)     | Attempt read, verify no incorrect mirror corruption       |
| Write UARTRSR_UARTECR without error condition     | Write any value, verify error flags remain at current state|
| Read UARTPeriphID/PCellID registers after write   | Verify values are unaffected by any write attempt         |

---

## 5. Verification Features

Each feature maps to a specific register/field from the RegMan.

| FID    | Register            | Field(s)              | Description                                                               | Verification Method          | Expected Result                                              |
|--------|---------------------|-----------------------|---------------------------------------------------------------------------|------------------------------|--------------------------------------------------------------|
| F-001  | UARTDR              | DATA[7:0]             | Write 8-bit data (TX), read 8-bit data (RX)                              | Frontdoor RW                 | Written value readable back; functionally drives TX/reads RX |
| F-002  | UARTDR              | OE, BE, PE, FE        | Error flags are hardware-set and software-readable only                   | Backdoor poke + frontdoor read | Flags reflect hardware error conditions; SW write has no effect |
| F-003  | UARTDR              | Reserved[15:12]       | Reserved field reads as 0                                                 | Frontdoor read               | Bits [15:12] = 0                                             |
| F-004  | UARTRSR_UARTECR     | OE, BE, PE, FE        | Read error status bits                                                    | Backdoor poke + frontdoor read | Error bits reflect injected hardware conditions              |
| F-005  | UARTRSR_UARTECR     | Clear_Errors[7:0]     | Write clears all error flags (OE, BE, PE, FE)                            | Frontdoor write + read RSR   | All error flags cleared after any write                      |
| F-006  | UARTRSR_UARTECR     | Reserved[7:4]         | Reserved field behavior (unpredictable per spec)                          | Frontdoor read               | No exception generated; value not checked                    |
| F-007  | UARTFR              | TXFE                  | TX FIFO empty flag, resets to 1                                           | Reset verify + frontdoor read | TXFE=1 after reset                                           |
| F-008  | UARTFR              | RXFE                  | RX FIFO empty flag, resets to 1                                           | Reset verify + frontdoor read | RXFE=1 after reset                                           |
| F-009  | UARTFR              | RXFF, TXFF            | FIFO full flags, reset to 0                                               | Reset verify + frontdoor read | Both = 0 after reset                                         |
| F-010  | UARTFR              | BUSY                  | UART busy flag set by hardware during TX                                   | Backdoor poke + frontdoor read | BUSY reflects TX activity                                    |
| F-011  | UARTFR              | CTS, DSR, DCD, RI     | Modem status bits (complement of modem inputs)                            | Backdoor poke + frontdoor read | Bits complement injected modem input signals                 |
| F-012  | UARTFR              | Reserved[15:9]        | Reserved field reads as zero                                               | Frontdoor read               | Bits [15:9] = 0                                              |
| F-013  | UARTILPR            | ILPDVSR[7:0]          | IrDA low-power divisor programmable 0x00–0xFF                             | Frontdoor RW                 | Written value readable back                                  |
| F-014  | UARTIBRD            | BAUD_DIVINT[15:0]     | Integer baud rate divisor, resets to 0                                    | Frontdoor RW + reset verify  | 0 after reset; stores and returns written values             |
| F-015  | UARTFBRD            | BAUD_DIVFRAC[5:0]     | Fractional baud rate divisor, resets to 0                                 | Frontdoor RW + reset verify  | 0 after reset; stores and returns written values             |
| F-016  | UARTLCR_H           | WLEN[6:5]             | Word length: all 4 encoding values (0b00–0b11)                            | Frontdoor RW (walk)          | Each written encoding readable back                          |
| F-017  | UARTLCR_H           | FEN                   | FIFO enable control                                                        | Frontdoor RW                 | 0=character mode, 1=FIFO mode; writable and readable         |
| F-018  | UARTLCR_H           | STP2                  | Two stop bits select                                                       | Frontdoor RW                 | Written value readable back                                  |
| F-019  | UARTLCR_H           | PEN, EPS, SPS         | Parity configuration fields                                                | Frontdoor RW                 | Written values readable back                                 |
| F-020  | UARTLCR_H           | BRK                   | Send break bit                                                             | Frontdoor RW                 | Written value readable back                                  |
| F-021  | UARTLCR_H           | Reserved[15:8]        | Reserved field reads as zero                                               | Frontdoor read               | Bits [15:8] = 0                                              |
| F-022  | UARTCR              | UARTEN                | UART enable, resets to 0                                                   | Frontdoor RW + reset verify  | 0 after reset; writable                                      |
| F-023  | UARTCR              | TXE                   | Transmit enable, resets to 1                                               | Frontdoor RW + reset verify  | 1 after reset; writable                                      |
| F-024  | UARTCR              | RXE                   | Receive enable, resets to 1                                                | Frontdoor RW + reset verify  | 1 after reset; writable                                      |
| F-025  | UARTCR              | LBE                   | Loopback enable, resets to 0                                               | Frontdoor RW + reset verify  | 0 after reset; writable                                      |
| F-026  | UARTCR              | SIREN, SIRLP          | SIR enable and low-power mode                                              | Frontdoor RW                 | Written values readable back                                 |
| F-027  | UARTCR              | CTSEn, RTSEn          | CTS/RTS hardware flow control enable                                       | Frontdoor RW                 | Written values readable back                                 |
| F-028  | UARTCR              | RTS, DTR, Out1, Out2  | Modem control outputs                                                      | Frontdoor RW                 | Written values readable back                                 |
| F-029  | UARTCR              | Reserved[6:3]         | Reserved field reads as zero                                               | Frontdoor read               | Bits [6:3] = 0                                               |
| F-030  | UARTIFLS            | TXIFLSEL[2:0]         | TX FIFO level: valid values 0b000–0b100, reset=0b010                       | Frontdoor RW + reset verify  | Reset=0b010; all legal values writable and readable          |
| F-031  | UARTIFLS            | RXIFLSEL[5:3]         | RX FIFO level: valid values 0b000–0b100, reset=0b010                       | Frontdoor RW + reset verify  | Reset=0b010; all legal values writable and readable          |
| F-032  | UARTIFLS            | Reserved[15:6]        | Reserved field reads as zero                                               | Frontdoor read               | Bits [15:6] = 0                                              |
| F-033  | UARTIMSC            | OEIM,BEIM,PEIM,FEIM   | Error interrupt masks, reset=0                                             | Frontdoor RW + reset verify  | 0 after reset; individually set/clear via write              |
| F-034  | UARTIMSC            | RTIM,TXIM,RXIM        | Data path interrupt masks, reset=0                                         | Frontdoor RW + reset verify  | 0 after reset; individually set/clear via write              |
| F-035  | UARTIMSC            | DSRMIM,DCDMIM,CTSMIM,RIMIM | Modem interrupt masks, reset=0                                        | Frontdoor RW + reset verify  | 0 after reset; individually set/clear via write              |
| F-036  | UARTRIS             | All status fields      | Raw interrupt status (hardware-driven, RO)                                | Backdoor poke + frontdoor read | Status bits reflect hardware-injected interrupt conditions   |
| F-037  | UARTMIS             | All masked status fields | Masked interrupt status = UARTRIS AND UARTIMSC                          | Frontdoor: set IMSC, inject via backdoor | UARTMIS reflects gated interrupts correctly              |
| F-038  | UARTICR             | All WO clear fields    | Writing 1 clears the corresponding UARTRIS/UARTMIS bit                    | Frontdoor write + read RIS/MIS | Cleared bits return to 0 in UARTRIS and UARTMIS           |
| F-039  | UARTDMACR           | TXDMAE, RXDMAE         | DMA enable fields, reset=0                                                | Frontdoor RW + reset verify  | 0 after reset; writable                                      |
| F-040  | UARTDMACR           | DMAONERR               | DMA-on-error, reset=0                                                     | Frontdoor RW                 | Written value readable back                                  |
| F-041  | UARTTCR             | ITEN                   | Integration test enable, reset=0                                           | Frontdoor RW                 | Written value readable back                                  |
| F-042  | UARTTCR             | TESTFIFO               | Test FIFO enable, reset=0                                                  | Frontdoor RW                 | Written value readable back                                  |
| F-043  | UARTTCR             | SIRTEST                | SIR test enable, reset=0                                                   | Frontdoor RW                 | Written value readable back                                  |
| F-044  | UARTITIP            | All RW fields          | Integration test input fields in test mode                                 | Frontdoor RW (ITEN=1)        | Written values readable back under integration test mode     |
| F-045  | UARTITOP            | All RW fields          | Integration test output fields                                             | Frontdoor RW (ITEN=1)        | Written values readable back                                 |
| F-046  | UARTTDR             | DATA[10:0]             | Test data register (TESTFIFO=1 required for operation)                    | Frontdoor RW (TESTFIFO=1)    | Written value readable back under test FIFO mode             |
| F-047  | UARTPeriphID0       | PartNumber0[7:0]       | Constant 0x11                                                              | Frontdoor read               | Reads back 0x11                                              |
| F-048  | UARTPeriphID1       | Designer0[7:4]=0x1, PartNumber1[3:0]=0x0 | Constant identification fields                          | Frontdoor read               | Designer0=0x1, PartNumber1=0x0                               |
| F-049  | UARTPeriphID2       | Revision[7:4]=0x3, Designer1[3:0]=0x4   | Revision-specific constant (r1p5)                      | Frontdoor read               | Revision=0x3, Designer1=0x4                                  |
| F-050  | UARTPeriphID3       | Configuration[7:0]=0x00 | Constant 0x00                                                             | Frontdoor read               | Reads back 0x00                                              |
| F-051  | UARTPCellID0        | UARTPCellID0[7:0]=0x0D | PrimeCell constant                                                         | Frontdoor read               | Reads back 0x0D                                              |
| F-052  | UARTPCellID1        | UARTPCellID1[7:0]=0xF0 | PrimeCell constant                                                         | Frontdoor read               | Reads back 0xF0                                              |
| F-053  | UARTPCellID2        | UARTPCellID2[7:0]=0x05 | PrimeCell constant                                                         | Frontdoor read               | Reads back 0x05                                              |
| F-054  | UARTPCellID3        | UARTPCellID3[7:0]=0xB1 | PrimeCell constant                                                         | Frontdoor read               | Reads back 0xB1                                              |

---

## 6. Test Plan

### 6.1 Frontdoor Read/Write Tests

| Test ID       | Test Name                          | Register(s)             | Description                                                                 | Pass Criteria                                            |
|---------------|------------------------------------|-------------------------|-----------------------------------------------------------------------------|----------------------------------------------------------|
| TC-FD-001     | uart_reg_hw_reset_test             | All (26 registers)      | Apply reset; frontdoor read all registers; compare against reset values     | All fields match RegMan-specified reset values           |
| TC-FD-002     | uart_reg_rw_wr_rd_test             | All RW registers        | Write 0xAA pattern, readback; write 0x55 pattern, readback                 | Readback matches written value for all RW fields         |
| TC-FD-003     | uart_reg_ro_write_no_effect_test   | UARTFR, UARTRIS, UARTMIS| Write all-1s to RO registers; read back                                     | Values unchanged (hardware or reset value preserved)     |
| TC-FD-004     | uart_reg_ro_field_in_rw_reg_test   | UARTDR, UARTLCR_H, UARTCR | Write to known RO fields in RW registers; read back                       | RO fields retain hardware-driven or reset value          |
| TC-FD-005     | uart_reg_uartdr_data_rw_test       | UARTDR                  | Write DATA[7:0] field; read back; verify error flag fields not writable     | DATA field readable; OE/BE/PE/FE not writable            |
| TC-FD-006     | uart_reg_baud_rate_rw_test         | UARTIBRD, UARTFBRD      | Walk integer and fractional divisor through all bit patterns                | All written values match readback                        |
| TC-FD-007     | uart_reg_lcr_h_wlen_walk_test      | UARTLCR_H               | Write WLEN = 0b00, 0b01, 0b10, 0b11; read each back                       | All four WLEN encodings preserved                        |
| TC-FD-008     | uart_reg_lcr_h_full_rw_test        | UARTLCR_H               | Write all RW fields; read back                                              | All writable fields match expected values                |
| TC-FD-009     | uart_reg_cr_defaults_test          | UARTCR                  | After reset, verify RXE=1, TXE=1, all others=0                             | RXE=1, TXE=1, UARTEN=0, LBE=0, SIREN=0 post-reset       |
| TC-FD-010     | uart_reg_cr_full_rw_test           | UARTCR                  | Write and read all RW fields                                                | All writable fields match expected values                |
| TC-FD-011     | uart_reg_ifls_reset_test           | UARTIFLS                | After reset, verify TXIFLSEL=0b010, RXIFLSEL=0b010                         | Both fields = 0b010 (1/2 full) post-reset                |
| TC-FD-012     | uart_reg_ifls_fifo_level_walk_test | UARTIFLS                | Walk TXIFLSEL and RXIFLSEL through 0b000–0b100                              | Each legal value preserved on readback                   |
| TC-FD-013     | uart_reg_imsc_set_clear_test       | UARTIMSC                | Set each mask bit individually; clear each individually; verify readback    | Each bit set/cleared independently                       |
| TC-FD-014     | uart_reg_icr_write_clears_ris_test | UARTICR, UARTRIS        | Inject interrupt condition via backdoor; write 1 to ICR bit; read RIS       | Corresponding UARTRIS bit clears after UARTICR write     |
| TC-FD-015     | uart_reg_icr_write_clears_mis_test | UARTICR, UARTMIS, UARTIMSC | Enable mask in IMSC; inject interrupt; write ICR; read MIS                | Corresponding UARTMIS bit clears after UARTICR write     |
| TC-FD-016     | uart_reg_dmacr_rw_test             | UARTDMACR               | Write/read RXDMAE, TXDMAE, DMAONERR                                         | All three fields writable and readable                   |
| TC-FD-017     | uart_reg_tcr_rw_test               | UARTTCR                 | Write/read ITEN, TESTFIFO, SIRTEST fields                                   | All three fields writable and readable                   |
| TC-FD-018     | uart_reg_peripid_constant_test     | UARTPeriphID0–3         | Frontdoor read all four ID registers                                        | PartNumber0=0x11, Designer0=0x1, Revision=0x3, etc.      |
| TC-FD-019     | uart_reg_pcellid_constant_test     | UARTPCellID0–3          | Frontdoor read all four PCell ID registers                                  | 0x0D, 0xF0, 0x05, 0xB1 respectively                     |
| TC-FD-020     | uart_reg_rsr_ecr_dual_access_test  | UARTRSR_UARTECR         | Inject error via backdoor; read error flags; write any value; read again    | Read shows error flags; after write all error flags = 0  |
| TC-FD-021     | uart_reg_ilpr_rw_test              | UARTILPR                | Write 0x00, 0xFF, 0xA5, 0x5A to ILPDVSR; read each back                   | All values match readback                                |

### 6.2 Backdoor Peek/Poke Tests

| Test ID       | Test Name                            | Register(s)         | Description                                                           | Pass Criteria                                    |
|---------------|--------------------------------------|---------------------|-----------------------------------------------------------------------|--------------------------------------------------|
| TC-BD-001     | uart_reg_backdoor_peek_reset_test    | All registers       | After reset, use peek() to read all register values                   | All fields match reset values in RegMan          |
| TC-BD-002     | uart_reg_backdoor_poke_rw_test       | All RW registers    | Use poke() to set fields; use read() to verify via frontdoor          | Frontdoor read returns value matching poke       |
| TC-BD-003     | uart_reg_backdoor_error_inject_test  | UARTDR, UARTRSR     | Poke OE/BE/PE/FE via backdoor; frontdoor read status                  | Error flags visible via frontdoor read           |
| TC-BD-004     | uart_reg_backdoor_fr_status_test     | UARTFR              | Poke TXFE, RXFE, BUSY, CTS etc. via backdoor; frontdoor read          | Flag register reflects poked state               |
| TC-BD-005     | uart_reg_mirror_consistency_test     | All RW registers    | Write via frontdoor; peek via backdoor; compare to RAL mirror value   | Mirror, DUT, and backdoor peek all agree         |

### 6.3 Reset Value Verification Tests

| Test ID       | Test Name                             | Registers                   | Description                                                              | Pass Criteria                                        |
|---------------|---------------------------------------|----------------------------|--------------------------------------------------------------------------|------------------------------------------------------|
| TC-RST-001    | uart_reg_por_reset_test               | All                        | Power-on reset; read all registers; compare to specified reset values    | All fields match RegMan reset values                 |
| TC-RST-002    | uart_reg_post_write_reset_test        | All RW                     | Write non-zero values to all RW registers; apply reset; read all back   | All fields return to reset values after reset        |
| TC-RST-003    | uart_reg_ro_reset_value_test          | All RO                     | After reset, read all RO registers and RO fields                        | All RO fields match expected reset values            |
| TC-RST-004    | uart_reg_constant_id_reset_test       | PeriphID, PCellID          | Verify ID registers return correct constants on any access              | Constants unchanged regardless of reset assertion    |

### 6.4 Access Permission Verification Tests

| Test ID       | Test Name                              | Registers                        | Description                                                           | Pass Criteria                                              |
|---------------|----------------------------------------|----------------------------------|-----------------------------------------------------------------------|------------------------------------------------------------|
| TC-ACC-001    | uart_reg_ro_no_write_test              | UARTFR, UARTRIS, UARTMIS         | Write all-1s; verify fields return pre-write (hardware-driven) value  | No RO field value changes on write attempt                 |
| TC-ACC-002    | uart_reg_wo_write_no_readback_test     | UARTICR, UARTRSR_UARTECR.Clear  | Write to WO fields; attempt read; verify RAL does not return written  | WO write triggers correct side-effect; read returns N/A   |
| TC-ACC-003    | uart_reg_rw_field_access_test          | All RW fields                    | Write legal values; read back; confirm accuracy                       | All RW fields return written values                        |
| TC-ACC-004    | uart_reg_reserved_zero_test            | Registers with "read as zero" reserved bits | Read reserved fields; verify zero                          | Reserved bits read as 0                                    |
| TC-ACC-005    | uart_reg_reserved_unpredictable_test   | UARTRSR[7:4], UARTTCR[15:3], UARTITIP[15:8], UARTTDR[15:11] | Read; verify no exception/error  | No simulation error; value not asserted                    |

### 6.5 Mirror/Update Operation Tests

| Test ID       | Test Name                         | Registers      | Description                                                                       | Pass Criteria                                          |
|---------------|-----------------------------------|----------------|-----------------------------------------------------------------------------------|--------------------------------------------------------|
| TC-MIR-001    | uart_reg_ral_mirror_update_test   | All RW         | Write via frontdoor; verify RAL mirror matches; call update(); verify DUT matches | Mirror and DUT stay in sync after write and update    |
| TC-MIR-002    | uart_reg_ral_set_get_test         | All RW         | Use set()/get() on RAL model without bus access; verify value in mirror           | RAL mirror holds set value before any bus transaction  |
| TC-MIR-003    | uart_reg_ral_predict_test         | All            | Force DUT state via backdoor poke; call predict(); verify RAL mirror matches      | RAL mirror updated to match forced DUT state           |
| TC-MIR-004    | uart_reg_ral_randomize_test       | All RW         | Randomize RAL fields; call update(); frontdoor read to verify                     | DUT register values match randomized RAL model state   |

### 6.6 Random Register Access Tests

| Test ID       | Test Name                          | Registers   | Description                                                                        | Pass Criteria                                          |
|---------------|------------------------------------|-------------|------------------------------------------------------------------------------------|--------------------------------------------------------|
| TC-RAND-001   | uart_reg_random_write_read_test    | All RW      | 1000 randomized write-then-read operations across all RW registers                 | All readbacks match written values                     |
| TC-RAND-002   | uart_reg_random_field_access_test  | All         | Randomize field-level writes; read register; compare field-by-field                | Each field independently holds correct value           |
| TC-RAND-003   | uart_reg_concurrent_rw_test        | All RW      | Interleaved writes and reads to different registers in random order                 | No data corruption between concurrent register accesses|

### 6.7 Boundary Value Tests

| Test ID       | Test Name                          | Register / Field              | Values Tested                                      | Pass Criteria                   |
|---------------|------------------------------------|-------------------------------|----------------------------------------------------|---------------------------------|
| TC-BV-001     | uart_reg_baud_divint_boundary_test | UARTIBRD.BAUD_DIVINT          | 0x0000, 0x0001, 0xFFFE, 0xFFFF                     | All values readable back        |
| TC-BV-002     | uart_reg_baud_divfrac_boundary_test| UARTFBRD.BAUD_DIVFRAC         | 0x00, 0x01, 0x3E, 0x3F                             | All values readable back        |
| TC-BV-003     | uart_reg_ilpdvsr_boundary_test     | UARTILPR.ILPDVSR              | 0x00, 0x01, 0xFE, 0xFF                             | All values readable back        |
| TC-BV-004     | uart_reg_wlen_boundary_test        | UARTLCR_H.WLEN                | 0b00, 0b01, 0b10, 0b11                             | All encodings readable back     |
| TC-BV-005     | uart_reg_fifo_level_boundary_test  | UARTIFLS.TXIFLSEL/RXIFLSEL    | 0b000, 0b001, 0b010, 0b011, 0b100                  | All legal values readable back  |
| TC-BV-006     | uart_reg_tdr_data_boundary_test    | UARTTDR.DATA                  | 0x000, 0x001, 0x7FE, 0x7FF (11-bit)               | All values readable back        |

### 6.8 Negative Tests

| Test ID       | Test Name                             | Register / Field               | Description                                                                  | Pass Criteria                                            |
|---------------|---------------------------------------|--------------------------------|------------------------------------------------------------------------------|----------------------------------------------------------|
| TC-NEG-001    | uart_reg_write_ro_register_test       | UARTFR, UARTRIS, UARTMIS       | Write to fully RO registers; read back                                       | No exception; field values unchanged                     |
| TC-NEG-002    | uart_reg_write_ro_field_test          | UARTDR[11:8], UARTLCR_H[15:8] | Write to RO fields within RW registers                                       | RO fields retain hardware or reset value                 |
| TC-NEG-003    | uart_reg_reserved_fifo_level_test     | UARTIFLS RXIFLSEL/TXIFLSEL     | Write reserved encodings 0b101, 0b110, 0b111                                | No hang/exception; no corruption of other fields         |
| TC-NEG-004    | uart_reg_write_peripid_test           | UARTPeriphID0–3, UARTPCellID0–3| Write arbitrary values to RO ID registers                                    | Constant values unchanged; no exception                  |
| TC-NEG-005    | uart_reg_icr_read_test                | UARTICR WO fields              | Attempt to read the WO ICR fields                                            | No meaningful data returned; no simulation error         |
| TC-NEG-006    | uart_reg_rsr_write_no_error_test      | UARTRSR_UARTECR                | Write any value when no error condition exists; read RSR error flags          | Error flags remain 0 (no spurious error set)             |

---

## 7. Functional Coverage Plan

### 7.1 Register Coverage

| Covergroup               | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| cg_reg_accessed          | Ensure every register has been accessed at least once via frontdoor      |
| cg_reg_written           | Ensure every writable register has been written at least once            |
| cg_reg_read              | Ensure every readable register has been read at least once               |
| cg_reg_reset_checked     | Ensure reset value has been verified for every register                  |

### 7.2 Field Coverage

For each register, per-field coverpoints must be defined:

| Covergroup Pattern       | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| cg_field_values          | Each 1-bit field must hit both 0 and 1                                   |
| cg_field_wlen            | UARTLCR_H.WLEN must hit all four values: 0b00, 0b01, 0b10, 0b11        |
| cg_field_rxiflsel        | RXIFLSEL must hit all five legal values: 0b000–0b100                    |
| cg_field_txiflsel        | TXIFLSEL must hit all five legal values: 0b000–0b100                    |
| cg_field_baud_divint     | BAUD_DIVINT must hit: 0x0000, mid-range, 0xFFFF                         |
| cg_field_baud_divfrac    | BAUD_DIVFRAC must hit: 0x00, 0x3F, and at least two intermediate values |
| cg_field_imsc_individual | Each of the 11 UARTIMSC mask bits must be individually exercised        |
| cg_field_icr_individual  | Each of the 11 UARTICR clear bits must be individually exercised         |

### 7.3 Access Type Coverage

| Covergroup               | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| cg_access_rw             | Every RW field exercised with at least one write and one read            |
| cg_access_ro_write_tried | Every RO register/field subjected to a write attempt (for negative test) |
| cg_access_wo_write       | Every WO field written at least once with side-effect verification       |
| cg_access_backdoor_peek  | Every readable register accessed via backdoor peek at least once         |
| cg_access_backdoor_poke  | Every writable register accessed via backdoor poke at least once         |

### 7.4 Reset Value Coverage

| Covergroup               | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| cg_reset_all_regs        | Every register's reset values verified after at least one reset event   |
| cg_reset_non_zero        | Specifically verify UARTCR.RXE=1, UARTCR.TXE=1, UARTFR.TXFE=1, UARTFR.RXFE=1, UARTIFLS.RXIFLSEL=0b010, UARTIFLS.TXIFLSEL=0b010, all PeriphID and PCellID constants |

### 7.5 Frontdoor vs. Backdoor Coverage

| Covergroup               | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| cg_fd_bd_cross           | Cross coverage of frontdoor and backdoor access to the same register     |
| cg_fd_bd_consistency     | Verify that after backdoor poke, frontdoor read returns matching value (cross-check) |

### 7.6 Interrupt Coverage

| Covergroup               | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| cg_imsc_all_bits         | All 11 interrupt mask bits in UARTIMSC individually set and cleared      |
| cg_ris_all_bits          | All 11 raw interrupt status bits in UARTRIS individually set             |
| cg_mis_gating            | UARTMIS correctly reflects each bit when IMSC mask=1 and RIS=1          |
| cg_icr_all_bits          | All 11 ICR clear bits individually exercised and verified to clear RIS   |

---

## 8. Requirements Traceability Matrix (RTM)

| Req ID   | Requirement Description                              | Register / Field             | Planned Test(s)                        | Verification Method        |
|----------|------------------------------------------------------|------------------------------|----------------------------------------|----------------------------|
| REQ-001  | UARTDR.DATA supports 8-bit TX/RX                     | UARTDR.DATA[7:0]             | TC-FD-005                              | Frontdoor RW               |
| REQ-002  | UARTDR error flags are hardware-set, SW-RO           | UARTDR.OE/BE/PE/FE           | TC-FD-004, TC-BD-003                   | Frontdoor + Backdoor       |
| REQ-003  | UARTRSR read returns error status, write clears errors| UARTRSR_UARTECR all fields   | TC-FD-020, TC-BD-003                   | Frontdoor dual-access      |
| REQ-004  | UARTFR.TXFE=1 and RXFE=1 at reset                    | UARTFR.TXFE, RXFE            | TC-FD-001, TC-RST-001                  | Reset verify               |
| REQ-005  | UARTFR is fully read-only                            | UARTFR (all fields)          | TC-ACC-001, TC-NEG-001                 | Frontdoor + Negative       |
| REQ-006  | UARTILPR.ILPDVSR is full 8-bit RW                    | UARTILPR.ILPDVSR             | TC-FD-021, TC-BV-003                   | Frontdoor RW + Boundary    |
| REQ-007  | UARTIBRD.BAUD_DIVINT is 16-bit RW, resets to 0       | UARTIBRD.BAUD_DIVINT         | TC-FD-006, TC-BV-001, TC-RST-001      | Frontdoor RW + Reset       |
| REQ-008  | UARTFBRD.BAUD_DIVFRAC is 6-bit RW, resets to 0       | UARTFBRD.BAUD_DIVFRAC        | TC-FD-006, TC-BV-002, TC-RST-001      | Frontdoor RW + Reset       |
| REQ-009  | UARTLCR_H.WLEN encodes 5/6/7/8 data bits             | UARTLCR_H.WLEN               | TC-FD-007, TC-BV-004                   | Frontdoor walk             |
| REQ-010  | UARTLCR_H.FEN controls FIFO vs character mode        | UARTLCR_H.FEN                | TC-FD-008                              | Frontdoor RW               |
| REQ-011  | UARTLCR_H parity fields (PEN, EPS, SPS) are RW       | UARTLCR_H.PEN/EPS/SPS        | TC-FD-008                              | Frontdoor RW               |
| REQ-012  | UARTCR.UARTEN=0 at reset                             | UARTCR.UARTEN                | TC-FD-009, TC-RST-001                  | Reset verify               |
| REQ-013  | UARTCR.RXE=1, TXE=1 at reset                         | UARTCR.RXE, TXE              | TC-FD-009, TC-RST-001                  | Reset verify               |
| REQ-014  | UARTCR.LBE enables loopback mode                     | UARTCR.LBE                   | TC-FD-010                              | Frontdoor RW               |
| REQ-015  | UARTCR modem controls (RTS, DTR, Out1, Out2) are RW  | UARTCR.RTS/DTR/Out1/Out2     | TC-FD-010                              | Frontdoor RW               |
| REQ-016  | UARTCR flow control (CTSEn, RTSEn) is RW             | UARTCR.CTSEn/RTSEn           | TC-FD-010                              | Frontdoor RW               |
| REQ-017  | UARTIFLS.RXIFLSEL and TXIFLSEL reset to 0b010        | UARTIFLS.RXIFLSEL/TXIFLSEL   | TC-FD-011, TC-RST-001                  | Reset verify               |
| REQ-018  | UARTIFLS FIFO levels accept 5 legal encodings        | UARTIFLS.RXIFLSEL/TXIFLSEL   | TC-FD-012, TC-BV-005                   | Frontdoor walk             |
| REQ-019  | UARTIMSC masks individually control each interrupt   | UARTIMSC all fields          | TC-FD-013                              | Frontdoor RW               |
| REQ-020  | UARTRIS reflects raw interrupt status (RO)           | UARTRIS all fields           | TC-FD-003, TC-BD-004, TC-ACC-001       | Backdoor + Frontdoor       |
| REQ-021  | UARTMIS = UARTRIS AND UARTIMSC                       | UARTMIS, UARTRIS, UARTIMSC   | TC-FD-015                              | Frontdoor gating check     |
| REQ-022  | UARTICR write 1 clears corresponding UARTRIS bit     | UARTICR, UARTRIS             | TC-FD-014                              | Frontdoor WO + read RIS    |
| REQ-023  | UARTDMACR.TXDMAE/RXDMAE enable DMA FIFOs            | UARTDMACR.TXDMAE/RXDMAE     | TC-FD-016                              | Frontdoor RW               |
| REQ-024  | UARTDMACR.DMAONERR disables DMA on error             | UARTDMACR.DMAONERR           | TC-FD-016                              | Frontdoor RW               |
| REQ-025  | UARTTCR.ITEN places UART in integration test mode    | UARTTCR.ITEN                 | TC-FD-017                              | Frontdoor RW               |
| REQ-026  | UARTTCR.TESTFIFO enables direct FIFO access          | UARTTCR.TESTFIFO             | TC-FD-017, TC-FD-021 (UARTTDR)        | Frontdoor RW               |
| REQ-027  | UARTITIP inputs readable/writable in test mode       | UARTITIP all fields          | TC-FD-021 (F-044)                      | Frontdoor RW (ITEN=1)      |
| REQ-028  | UARTITOP outputs readable/writable in test mode      | UARTITOP all fields          | TC-FD-021 (F-045)                      | Frontdoor RW (ITEN=1)      |
| REQ-029  | UARTTDR.DATA 11-bit test data register               | UARTTDR.DATA[10:0]           | TC-BV-006, TC-FD-017                   | Frontdoor RW (TESTFIFO=1)  |
| REQ-030  | UARTPeriphID0.PartNumber0 = 0x11                     | UARTPeriphID0.PartNumber0    | TC-FD-018                              | Frontdoor read constant    |
| REQ-031  | UARTPeriphID1: Designer0=0x1, PartNumber1=0x0        | UARTPeriphID1                | TC-FD-018                              | Frontdoor read constant    |
| REQ-032  | UARTPeriphID2: Revision=0x3 (r1p5), Designer1=0x4   | UARTPeriphID2                | TC-FD-018                              | Frontdoor read constant    |
| REQ-033  | UARTPeriphID3.Configuration = 0x00                   | UARTPeriphID3                | TC-FD-018                              | Frontdoor read constant    |
| REQ-034  | UARTPCellID0 = 0x0D                                  | UARTPCellID0                 | TC-FD-019                              | Frontdoor read constant    |
| REQ-035  | UARTPCellID1 = 0xF0                                  | UARTPCellID1                 | TC-FD-019                              | Frontdoor read constant    |
| REQ-036  | UARTPCellID2 = 0x05                                  | UARTPCellID2                 | TC-FD-019                              | Frontdoor read constant    |
| REQ-037  | UARTPCellID3 = 0xB1                                  | UARTPCellID3                 | TC-FD-019                              | Frontdoor read constant    |

---

## 9. Regression Plan

### 9.1 Test Grouping

| Group        | Tests Included                                             | Run Frequency  | Purpose                                       |
|--------------|------------------------------------------------------------|----------------|-----------------------------------------------|
| SMOKE        | TC-FD-001, TC-RST-001, TC-FD-009, TC-FD-018, TC-FD-019   | Every commit   | Quick sanity: reset, CR defaults, ID regs     |
| REGRESSION   | All TC-FD-*, TC-RST-*, TC-ACC-*, TC-NEG-*                  | Nightly        | Full frontdoor register verification          |
| FULL         | All tests including TC-BD-*, TC-MIR-*, TC-BV-*, TC-RAND-* | Weekly         | Complete register closure including backdoor  |
| RANDOM       | TC-RAND-001, TC-RAND-002, TC-RAND-003 (multiple seeds)     | Nightly        | Random stimulus corner case discovery         |

### 9.2 Seed Strategy

- Nightly regression: 10 unique seeds per random test
- Weekly full regression: 50 unique seeds per random test
- Coverage closure: additional seeds until coverage goals are met

### 9.3 Tool Requirements

| Tool Category         | Requirement                                                      |
|-----------------------|------------------------------------------------------------------|
| Simulator             | SystemVerilog UVM-capable simulator (Information Required)       |
| Coverage Tool         | Functional coverage merge and analysis tool (Information Required)|
| RAL Generator         | UVM RAL model generated from ARM_UART.regman                    |
| APB VIP               | APB3-compliant UVM VIP for frontdoor transactions                |
| HDL Backdoor Access   | Simulation-compatible HDL path access for peek/poke              |

---

## 10. Exit Criteria

### 10.1 Test Execution Criteria

| Criterion                           | Target     |
|-------------------------------------|------------|
| All planned tests pass              | 100%       |
| Zero P1/P2 open bugs                | 100%       |
| Zero timeout or simulation errors   | 100%       |
| All negative tests correctly reject | 100%       |

### 10.2 Functional Coverage Criteria

| Coverage Category                   | Target     |
|-------------------------------------|------------|
| Register access coverage            | 100%       |
| Field value coverage (all legal)    | 100%       |
| Reset value verification            | 100%       |
| RO write-protection verified        | 100%       |
| WO write side-effect verified       | 100%       |
| Backdoor peek coverage              | 100%       |
| Backdoor poke coverage              | 100%       |
| RAL mirror consistency              | 100%       |
| Interrupt mask/status/clear matrix  | 100%       |
| WLEN all 4 encodings                | 100%       |
| FIFO level all 5 legal encodings    | 100%       |
| ID register constants verified      | 100%       |

### 10.3 Sign-Off Requirements

| Sign-Off Item                                           | Responsible Party         |
|---------------------------------------------------------|---------------------------|
| All tests listed in Section 6 executed and passing      | DV Lead                   |
| Coverage goals in Section 10.2 met                      | DV Lead                   |
| RTM fully populated with test results                   | DV Lead                   |
| Bug database reviewed: all P1/P2 closed or waived       | Design + DV Leads         |
| Final regression run report reviewed and approved       | Project Lead              |

---

## Appendix A: Access Type Definitions

| Access Type | Abbreviation | Read Behavior                          | Write Behavior                          |
|-------------|--------------|----------------------------------------|-----------------------------------------|
| Read-Write  | RW           | Returns current field value            | Updates field value                     |
| Read-Only   | RO           | Returns hardware-driven or reset value | Write has no effect                     |
| Write-Only  | WO           | Not applicable (no read data returned) | Write triggers side-effect              |
| Default     | (HW access)  | Hardware can read the field            | Hardware uses field as configuration    |

## Appendix B: Register Access Summary

| Access Category       | Count | Register Names                                                                                                         |
|-----------------------|-------|------------------------------------------------------------------------------------------------------------------------|
| Fully RW              | 10    | UARTDR, UARTRSR_UARTECR (dual), UARTILPR, UARTIBRD, UARTFBRD, UARTLCR_H, UARTCR, UARTIFLS, UARTIMSC, UARTDMACR, UARTTCR, UARTITIP, UARTITOP, UARTTDR |
| Fully RO              | 10    | UARTFR, UARTRIS, UARTMIS, UARTPeriphID0–3, UARTPCellID0–3                                                             |
| Mixed (RW + WO)       | 1     | UARTICR (Reserved=RO, active fields=WO)                                                                                |
| Dual RO/WO            | 1     | UARTRSR_UARTECR (read returns status; write clears errors)                                                             |

## Appendix C: Non-Zero Reset Value Summary

| Register    | Field         | Reset Value | Decimal |
|-------------|---------------|-------------|---------|
| UARTFR      | TXFE          | 1           | 1       |
| UARTFR      | RXFE          | 1           | 1       |
| UARTCR      | RXE           | 1           | 1       |
| UARTCR      | TXE           | 1           | 1       |
| UARTIFLS    | RXIFLSEL      | 0b010       | 2       |
| UARTIFLS    | TXIFLSEL      | 0b010       | 2       |
| UARTPeriphID0 | PartNumber0 | 0b10001     | 0x11    |
| UARTPeriphID1 | Designer0   | 0b0001      | 0x1     |
| UARTPeriphID2 | Revision    | 0b0011      | 0x3     |
| UARTPeriphID2 | Designer1   | 0b0100      | 0x4     |
| UARTPCellID0  | UARTPCellID0| 0b00001101  | 0x0D    |
| UARTPCellID1  | UARTPCellID1| 0b11110000  | 0xF0    |
| UARTPCellID2  | UARTPCellID2| 0b00000101  | 0x05    |
| UARTPCellID3  | UARTPCellID3| 0b10110001  | 0xB1    |

---

*End of Document*
