# Fast Runner Game Explanation

This document explains the basic structure and gameplay of the 'Phasor Fast Runner' game, implemented in `static/fast-runner.html`.

## Introduction

'Fast Runner' is a simple 2D auto-runner game built using the Phaser 3 game framework. The player character automatically moves forward through a generated level, and the player's primary interaction is jumping to navigate obstacles and collect items.

## Gameplay

The core gameplay loop involves:
- **Auto-Running:** The player character automatically moves to the right at a constant speed.
- **Jumping:** The player can jump to clear gaps and avoid obstacles.
- **Collecting Rings:** Yellow rings are scattered throughout the level. Collecting them increases the player's score.
- **Avoiding Obstacles:** Red spikes are placed as hazards. Touching a spike results in a game over.
- **Reaching the Goal:** A green goal post marks the end of the level. Reaching it completes the level.

The game world is wider than the screen, and the camera follows the player horizontally.

## Controls

- **Up Arrow Key:** Jump
- **Left Arrow Key:** Move left (slightly slower than auto-run)
- **Right Arrow Key:** Move right (slightly faster than auto-run)

## Objectives

- **Win:** Reach the green goal post at the end of the level.
- **Lose:** Hit a red spike or fall off the bottom of the level.

## Scoring

Points are awarded for collecting yellow rings. Each ring collected adds 10 points to the player's score.

## Technical Details

The game is built using the Phaser 3 framework. Graphics for the player, platforms, rings, spikes, and goal are generated dynamically using Phaser's graphics capabilities rather than loading external image assets. The level structure, including platforms, gaps, rings, and spikes, is defined programmatically within the `create` function of the `GameScene`.