# Lithic Model — Why this skill uses geology and metallurgy

This reference explains the reasoning model used by `/cli-forge-oci-rootless`.
It is deliberately different from biological resilience metaphors used by other
skills. The goal is a new operational lens: material transformation under
constraints.

## Why geology/metallurgy fits this migration

An Ansible/bare-metal middleware migration is rarely a clean rewrite. It is more
like surveying a geological formation:

- layers accumulated over years
- valuable ore mixed with accidental waste
- faults hidden below the surface
- interfaces that crack under pressure
- old residues that contaminate the new system if unmanaged

The OCI target is an engineered alloy: runtime image, init logic, tools image,
check image, host bootstrap, Quadlet, systemd --user, and public CLI must combine
into one material with known properties.

## Concept map

| Lithic concept | Operational interpretation |
|---|---|
| Bedrock | Durable contract that must not move silently |
| Strata | Historical layers: Ansible, shell, systemd, VM conventions |
| Ore | Useful legacy behavior to refine |
| Gangue | Accidental mechanics to discard |
| Core sample | Evidence packet tied to file/path/line/command |
| Fault line | Boundary where rootless migration often breaks |
| Phase diagram | Stable/unstable combinations of OS, Podman, cgroups, SELinux, user bus, registry, storage |
| Alloy | Target OCI product assembled from specialized artifacts |
| Heat treatment | Controlled rollout, hardening, repeated rerun/reboot/degraded tests |
| Fracture surface | Failure mode revealed under stress |
| Tailings | Legacy residue that must be removed, contained, or sunset |
| Stratigraphic memory | Runbooks, blackbox entries, anti-regression tests |

## Design pressure this model creates

1. **Sample before designing.** Do not infer the contract from desired target
   files. Extract it from legacy evidence.
2. **Classify material.** Every legacy item is Preserve, Refine, Adapter,
   Tailings, or Unknown.
3. **Map faults.** Rootless migration risk concentrates at boundaries, not just
   inside images.
4. **Forge one alloy.** Avoid multiple control planes. Runtime, tools, checks,
   bootstrap, CLI, and Quadlet must have one responsibility map.
5. **Stress the material.** A green demo is not proof. Test repeated cycles,
   dirty reruns, reboot, failed secrets, corrupt backups, and monitoring bridge
   failures.
6. **Contain residues.** Legacy compatibility is temporary unless it is an
   adapter over the public CLI with removal conditions.

## Anti-patterns caught by the model

- Treating a Dockerfile as the whole migration.
- Baking init or maintenance tools into the runtime image without reason.
- Copying Ansible variables without deciding whether they are bedrock or strata.
- Assuming `podman run` proves `systemd --user` after reboot.
- Assuming local health proves remote monitoring.
- Claiming backup capability without restore proof.
- Leaving old wrappers that mutate state outside the new CLI.
- Keeping old systemd root units beside Quadlet user units.
- Leaving tailings undocumented because “they no longer matter”.
