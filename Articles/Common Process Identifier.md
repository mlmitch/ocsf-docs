# Common Process Identifier (CPID)

The Common Process Identifier (CPID pronounced "see-pid") is a specification for deterministically calculating an identifier that uniquely identifies an endpoint process across an organization.

This document introduces the CPID by:
1. Exposing the problem it solves
2. Elaborating on additional properties we want in a solution
3. Specify the exact calculation procedure

## Problem Exposition
[OCSF Proposal #1205](https://github.com/ocsf/ocsf-schema/discussions/1205) introduced this topic to the OCSF community with a brief problem explanation.
We'll dive a little deeper here.

### Unique Endpoint Process Identification
Unique endpoint process identification is a problem because major operating system families (Windows, MacOS, Linux) do not implement process identification that is rigorous enough for adversarial cybersecurity use cases.

The operating system process identifier (PID) is an integer, usually no larger than 32 bits, which is assigned to a process by the operating system at process creation time.
PIDs uniquely identify a process within a given endpoint during the lifetime of that process.

However, after a process terminates, the operating system can and will reuse the PID.
PID reuse is encountered frequently in normal circumstances depending on the operating system.
Furthermore, PID reuse can be exacerbated or induced.
An example of this would be a software bug that leaves many zombie processes running on a system.
This would shrink the available PID pool and result in more PID reuse.

Given that PIDs are reused, a popular technique to achieve unique process identification within a given endpoint is to combine the PID with the Process Creation Time (PCT).
A PID is unique for a process during its lifetime and the PCT identifies the start of that lifetime.
Therefore, PID and PCT provide unique identification across all time within that endpoint.

The next step is usually to add more identifying information to achieve organizationally or globally unique identification.
The most straightforward way to do this is to combine PID and PCT with some sort of unique endpoint identifier.
There are well-known examples of this approach like [Sysmon GUIDs](https://learn.microsoft.com/en-us/answers/questions/234884/structure-of-process-guids-used-in-sysmon-etw-even).
Hostname is a frequently used for this purpose, but it comes with issues.

Generic unique endpoint identification is not a solved problem.
Endpoint identifiers often don't end up being unique in practice due to system misconfiguration or virtualization complications.
Additionally, some endpoint identifiers can change through administrative action during the lifetime of a process, making process identification less reliable.
These issues encourage endpoint security software to rely on their own endpoint identification system.

Still, on some platforms there can be issues with approaching process identification using endpoint identity, PID and PCT.
PCT needs to be expressed as a "wall clock" value to achieve local uniqueness.
Depending on the operating system, administrative clock changes can affect the reporting of these values.

Given the above complications, many endpoint security software implementations choose to generate a random unique process identifier at process creation time.
There is no doubt that this approach achieves unique endpoint process identification.

### Shared Unique Endpoint Process Identification

The previous section describes how unique endpoint process identification can be achieved.
However, the two widely adopted implementation routes create identification schemes specific to the implementing software.
This disrupts downstream data consumers (e.g. SOC data analysis) when multiple endpoint security software implementations are present.
Here are a couple examples of when the problem arises:
- an organization runs 2 competing EDR products side by side
- an incident response team uses tooling in addition to their organization’s EDR solution
- datasets from different endpoint focus areas (EDR, DLP, Device Management, Identity) need to be correlated

In these scenarios, the data consumer must work through the unique endpoint process identification problem using whatever data they have on hand.
PID is a universal data point that is usually present in endpoint data sets.
PCT is usually present in EDRs, but the timestamp resolution reporting can vary in addition to the timestamp source.
Assuming these two datapoints are present and processable, then data consumers are usually still left relying on hostname for endpoint identification.
This can often be good enough depending on your datasets, but it is not a general solution, and the weaknesses show when considering many endpoints.

A comprehensive solution requires a process identification system shared by the different dataset producers.

It is theoretically possible for endpoint security software creators to implement a unique process identifier consensus mechanism.
Such a mechanism could facilitate using a randomly generated identifier that guarantees unique identification.
However, synchronous coordination between software creators and the implementation of such a consensus mechanism create significant barriers.

Ideally, operating systems would implement more rigorous process identification that could be consumed by endpoint security software.
While possible, it is unlikely that all major operating systems implement this.
Additionally, this would only be a solution for operating system versions after the addition of improved process identification.

We believe a good solution is possible by rigorously specifying how to deterministically arrive at a process identifier using only information provided by the operating system.
This still requires adoption by endpoint security software creators.
However, barriers are reduced because software creators can independently adopt the specification without coordination between creators.
Additionally, the technical implementation becomes simpler since no on-endpoint communication between different endpoint security software is required.

## Solution Properties

Before getting into the CPID specification, this section covers some additional properties that we want to see in a deterministic process identification scheme.

### Unique

Covered in depth above, endpoint processes should be uniquely identified.
Therefore, the process identification scheme should attempt to minimize the probability of random or systemic collisions.
There should be strong uniqueness guarantees for process identification within an organization.
Global uniqueness guarantees are desirable but not essential.
Note the trade-off between uniqueness and efficiency since larger identifiers are required to achieve greater uniqueness.

### Easy to Use

The identification scheme should be easy to use for identifying endpoint processes above all else.
There are multiple facets to this.

First, the identifier should have identical usage patterns regardless of the process's platform.
The process is a fundamental entity on all widely deployed operating systems.
Maintaining consistent experience across platforms enables data consumers to rely on the abstraction of an endpoint process instead of platform-specific details.

Second, the identifier should maximize its compatibility with software that could use it.
Making the identifier easy to consume in other software maximizes the value that data consumers can realize.

Related to maximizing compatibility is the privilege required to source identifier information from the host endpoint.
An identifier scheme requiring high-level privilege or access may be harder to adopt.
Therefore low-privilege data sources should be preferred where possible.

### Efficient

The identification scheme should be efficient both in terms of resources required for computation and the resulting identifier size.

The process identifier size can have a large impact on the datasets it is used with.
Modern endpoint security datasets are process focused.
There are events to describe the creation of endpoint processes.
Furthermore, other system activities are usually expressed in terms of an "actor" process that performed an action with the actor process referred to by its unique identifier.
We want to avoid unnecessarily bloating datasets since process identifiers are used so often.

The unique process identifier should also be efficiently calculatable and usable by endpoint security software.
Endpoint security software must operate in a manner that does not disrupt the host endpoint through excessive resource consumption.
Identifier size plays a role here as endpoint software will need to keep a set of process identifiers in memory.
However, the computational resources used to calculate the identifiers has the potential to be more disruptive.
Identifier calculation will need to occur during most process creations.
Depending on the system, normal operation could involve continuously spawning many processes (e.g. software build server).
Unique identifier calculation for every process should be enabled without adversely impacting system performance.

### Reliable

Identifiers should be reliably calculatable across the entire lifetime of a process.
This helps guarantee deterministic calculation across endpoint software that may attempt to calculate an identifier at different points in a process's lifetime.
Furthermore, we don't want the inputs to our identifier calculation to be (easily) alterable by attacks or other means.
Immutable inputs that are available for the lifetime of a process are best.
We should prefer inputs whose modification we can feasibly detect in the absence of immutable inputs.

### Safe

Process identifiers should be safe to rely on in downstream systems despite adversarial action.
An example of this is relying on unique process identifiers as a database primary key.
Depending on the identifier construction, it may be possible for an attacker to induce database performance issues through PID reuse or some other attack.
The identifier design should avoid these possibilities where possible.

### Private

Finally, our identifiers shouldn't leak information about the endpoint or process without good justification.
It is highly likely that datasets containing process identifiers will be shared.
The identifiers should seek to minimize the risk of undesired information exposure.

## CPID Specification
