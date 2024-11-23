# Reboot to apply kernel updates

## Goal

As part of my role maintaining systems in the microspace I created this playbook **placeholder** to automate the process of running daily updates against infrastructure and also sending notifications to a discord channel that I moniter. An important part of patching is updating Linux hosts to apply updated kernels.

## Execution

This playbook gets run manually by a system admin when they notice that one or several systems require a reboot. The playbook applys a serial restraint on the playbook to only execute reboots one system at a time with a 1 minute wait inbetween. While this probably doesn't scale very well out to hundreds of systems, for now it works well to ensure systems hosting critical services in a swarm are not targeted all at once. Since the infrastructure Im keeping an eye on is also used by others in the u-space and eventually will also include services at the makerspace and for cascadeSTEAM Im planning for maintaining services as much as possible.

## The code

The ansible script can be found here **placeholder**