---
title: "Protocols"
weight: 2
dashboardWeight: 0.2
dashboardState: incomplete
dashboardAudit: 0
---

# Protocol
---

## Key concepts

A miner has power based on the storage capacity they provide to the network.

The "`Power` actor" actor is a singleton actor that exists on the Filecoin VM. It manages power allocation and rewards.

The "quality adjusted power" or QAp of a sector is the amount of power assigned to a sector. A miner wins block reward proportionally to the total QAp of their sectors across their Miner actors.

The "power table" is what we informally refer to the table stored in the Power actor which reflects how much QA power each miner has.

## Mining Protocols

### Miner cycle
#### Register a miner
A user registers a new `Miner` actor via `Power.CreateMiner`.

#### Add storage capacity
A miner adds storage capacity to the network by adding sectors to a `Miner` actor.

- The miner collects storage deals and publish them via `Market.PublishStorageDeals`.
- The miner groups deals in a sector, runs the sealing process and submits the sealed sector information via `miner.PreCommitSector`. The sector state is now *precommitted*.
- The miner waits `PreCommitChallengeDelay` since the on-chain precommit epoch, gets the `InteractiveRandomness`, generates a Seal Proof and submits it via `miner.ProveCommitSector`. The sector state is now *committed*.

#### Mantain storage power

A miner mantains sectors *active* by timely submitting Proofs-of-Spacetime (PoSt).
A PoSt proves sectors are persistently stored through time.

A miner must timely submit `miner.SubmitWindowedPoSt` for their sectors according to their assigned deadlines (see deadline section).

### Deadlines

#### Proving Period
A *proving period* is a period of `WPoStProvingPeriod` epochs in which a `Miner` actor is scheduled to prove its storage.
A *proving period* is evenly divided in `WPoStPeriodDeadlines` *deadlines*.

Each miner has a different start of proving period `ProvingPeriodStart` that is assigned at `Power.CreateMiner`.

#### Deadline
A *deadline* is a period of `WPoStChallengeWindow` epochs that divides a proving period.
Sectors are assigned to a deadline on `miner.ProveCommitSector` and will remain assigned to it throughout their lifetime.
Sectors are also assigned to partitions.

There are four relevant epochs associated to a deadline.
- Open Epoch: Epoch from which a PoSt Proof for this deadline can be submitted.
- Close Epoch: Epoch after which a PoSt Proof for this deadline will be rejected.
- Fault Cutoff Epoch: Epoch after which a `DeclareFault` for sectors in the upcoming deadline are rejected.
- Challenge Epoch: Epoch at which the randomness for the challenges is available.

A miner must submit a `miner.SubmitWindowedPoSt` for each deadline.

#### Partitions
A partition is a set of sectors that is not larger than the Seal Proof `sp.WindowPoStPartitionSectors`.

Partitions are a by-product of our current proof mechanism. There is a limit of sectors (`sp.WindowPoStPartitionSectors`) that can be proven in a single SNARK proof. If more than this amount is required to be proven, more than one SNARK proof is required. Each SNARK proof is a partition.

Sectors are assigned to a partition at `miner.ProveCommitSector` and they can be re-arranged via `CompactPartitions`.

#### Deadline assignment

### Sectors

A *sector* is a storage container in which miners store deals from the market.

- `Precommitted`: miner seals sector and submits `miner.PreCommitSector`
- `Committed`: miner generates a Seal proof and submits `miner.ProveCommitSector`
- `Active`: miner generate valid PoSt proofs and timely submits `miner.SubmitWindowedPoSt``Committed`
- `Faulty`: miner fails to generate a proof (see Fault section)
- `Recovering`: miner declared a faulty sector via `miner.DeclareFaultRecovered`

### Storage Power

A miner wins blocks proportionally to their Quality Adjusted Power.

Sector quality is based on the size of deals in a sectors and the quality of these deals.

`$a$`

`$\frac{\mathsf{VerifiedDeadMultiplied} 2^\mathsf{SectorQualityPrecision}}{QualityBaseMultiplier}$`

#### Committed Capacity
A `Committed Capacity sector` is a sector with no deals. It can be early terminated via the upgrade protocol (see Sector Upgrading).

The power of this sector is its raw bytes multiplied by `QualityBaseMultiplier`.

#### Sector with Deals


## Faults and Penalties

### Collaterals
#### PreCommitDeposit

`PreCommitDeposit` is the collateral deposited on `miner.PreCommitSector` and it is returned on `miner.ProveCommit`. If the `miner.ProveCommit` is not executed before `MaxProveCommitDuration` epochs since precommit.

The `PreCommitDeposit` is locked separately from Initial Pledge and cannot be used as collateral for other faults.

#### InitialPledge
#### LockedFunds

### Fees
#### Fault Fee (FF)
Fault Fee (or "FF") = 2.15 BRapprox

#### Sector Penalization (SP)
Sector Penalization (or "SP") = 5 BRapprox

#### Termination Fee (TF)
Termination Fee (or "TF") = TODO

#### PreCommit Deposit Fee (PCD)
Sector Penalization (or "SP") = 20 BRapprox

#### Consensus Fault Fee (CFF)
Consensus Fault Fee (or "CFF") = 5 br

### Faults

A sector is considered faulty if a miner is not able to generate a proof.
There could be different reasons for which a sector can be faulty: power outage, hardware failure, internet connection loss.

If a sector state is not *active*, it will not count towards power.

#### PreCommit Faults
A sector fault is a "precommit fault" if it is *precommitted* after `MaxProveCommitDuration` epochs since precommit.

**Penalization**:
- Fee: PreCommitDeposit fee is charged on its first deadline. The fee is charged from the PreCommitDeposit collateral.
- Sector state: Sector is immediatedly marked as *deleted*.

#### Declared Faults
A sector fault is a "declared fault" if it is *faulty* on the deadline's fault cutoff epoch.

There are three cases of Declared Faults:
- **New Declared Fault**: A sector is *active*, and its Miner actor processes `miner.DeclareFault` before the deadline's fault cutoff epoch.
- **Declared Failed Recovery Fault**:  A sector is *recovering* and its Miner actor processes `miner.DeclareFault` before the deadline's fault cutoff epoch.
- **Continued Fault**: A sector was faulty in the previous proving period and continues beind so.

**Penalization:** 
- Fee: Fault fee is charged on its next deadline. The fee is charged from Initial Pledge.
- Sector state: Sector is immediatedly marked as *faulty*.

#### Skipped Faults
A sector fault is a "skipped fault" if the sector is not *faulty* before deadline's fault cutoff epoch, the miner submits a `miner.SubmittedWindowPoSt` marking the sector as skipped.

There two cases of Skipped Faults:
- **Active Skipped Fault**: A sector is *active* and the miner marked it as skipped at `miner.SubmittedWindowPoSt`.
- **Recovered Skipped Fault**: A sector is *recovering* and the miner marked it as skipped at `miner.SubmittedWindowPoSt`.

**Penalization**:
- Fee: Sector Penalization is immediatedly charged (on `miner.SubmittedWindowPoSt`). The fee is charged from Initial Pledge.
- Sector state: Sector is immediatedly marked as *faulty* (on `miner.SubmittedWindowPoSt`).

#### Detected Faults
A sector fault is a "detected fault" if the sector is not already *faulty* before deadline's fault cutoff epoch and no `miner.SubmittedWindowPoSt` is executed before the deadline's close epoch.

There two three of Skipped Faults:
- **Active Skipped Fault**: A sector is *active* and the miner missed `miner.SubmittedWindowPoSt`.
- **Recovered Skipped Fault**: A sector is *recovering* and the miner marked it as skipped at `miner.SubmittedWindowPoSt**.

**Penalization**:
- Fee: Sector Penalization is immediatedly charged (on `ProvingDeadline`). The fee is charged from Initial Pledge.
- Sector state: Sector is immediatedly marked as *faulty* (on `ProvingDeadline`).


## Rewards
