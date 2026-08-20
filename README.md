# Drone Skills

Claude skills for drone pilot workflows: photogrammetry, orthomosaics, and the processing
chains that follow a flight.

The rule for this repo is that quality beats quantity. Every skill here was tested against
a real workflow before it was published, and skills that need specific software or hardware
say which and why. If a skill is on the list, it works.

## Using a skill

Each skill is a directory containing a `SKILL.md`. Copy the directory into your Claude
skills folder (for Claude Code: `~/.claude/skills/`) and invoke it by name, or just ask
Claude for the task the skill covers.

## Status

Early days. First skill shipped: [orthomosaic processing](orthomosaic-processing/SKILL.md)
with OpenDroneMap — tested end to end against a real 172-image mapping flight.
More photogrammetry workflows in progress.

Maintained by [Nebli](https://nebli.ai).
