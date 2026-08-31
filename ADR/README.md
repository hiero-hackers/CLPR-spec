# CLPR ADR Guide

This document governs Architecture Decision Records (ADRs) for the CLPR protocol specification: what
a committed ADR should look like. 

## 1. Repository Convention

ADRs are named `YYYY-MM-DD-short-title.md`, dated so they sort chronologically. Only final-form
ADRs are committed into the main branch of the repo. Working material used to produce an ADR — notes, drafts, ontology breakdowns —
may be committed on PR drafts, but are never merged into the main branch.

## 2. The ADR Final Form

A final ADR consists of an Abstract followed by four numbered sections and two optional appendices.

### 2.1 Abstract

Positioned at the top of the document, before Section 1. The abstract states the need for the ADR and is  
followed by an enumeration of "**Changes in this ADR:**". Each entry in the enumeration
is a single short sentence that could function as a changelog top-line.

### 2.2 Section 1: Problem Description

Each discrete problem area the ADR addresses gets its own subsection, establishing the ADR's scope. A
problem statement identifies the current behavior, why it is inadequate, and the concrete consequence.

### 2.3 Section 2: Options Considered and Non-Goals

Options Considered subsection presents the major alternative solutions that were considered and rejected, 
motivating the accepted solution.  This is not an exhaustive list or low level description of 
design decisions, but provides high level or fundamental architectural considerations.

The Non-Goals subsection details considerations that are explicitly left out of the ADR as out of scope 
or in the domain of a different ADR. 

### 2.4 Section 3: The Solution

The complete, accepted solution: new or changed data structures, new or changed methods, new or
changed actor behavior. This section is the specification itself. Organization is chosen per ADR to
suit the content.

Every new or changed data structure is presented in whole. 
Two kinds of data structure are presented differently:

- **On-the-wire** (anything traversing sync between endpoints) is **literal protobuf**:
  `message X { type field = N; }`. The protobuf is the wire format, and every implementation matches
  it exactly.
- **On-chain and local service state** (records held in contract or node state) is a **platform-neutral
  struct**: `field : type // comment`, no field numbers, no `message` wrapper. This format is informational
  and does not correspond to any data format. Each platform adapts it to the concrete representation 
  (integer widths, storage layout) of its native format when implemented.

### 2.5 Section 4: Testing

The tests suggested to verify a correct implementation: happy-path capability tests, required failure
scenarios, and edge cases likely to be implemented incorrectly.

### 2.6 Appendix A: Patching the Spec

Literal file-and-line-number references for every change this ADR requires in the existing
specification text. This appendix is optional, and not useful until the ADR's design is stable.

### 2.7 Appendix B: Implementation Notes

Recommendations for implementing the ADR. This may include additional algorithms, design considerations, and 
notes on how different chains may implement the spec changes. 
