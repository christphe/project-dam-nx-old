# Changelog

## [Unreleased] - minor

### Added
- Aim sensitivity on the Controller page: how fast the right stick turns while aiming

### Changed
- The options no longer show the Mouse and Key Bindings pages, which need a mouse or a keyboard; Aim sensitivity replaces the aim-mode sensitivity and precision curve options
- Aim style moves to the Controller page and is Centered by default: the right stick turns the camera and the view stays where it is when the stick is released (Original keeps the N64 crosshair)
- Faster rendering of busy scenes: vertex data goes to the GPU through one streaming buffer instead of a new buffer per draw, about three times cheaper per draw on the console

### Fixed
- Restart Game no longer shows at the bottom of the options menu on the Switch, where it could only quit the game

## [0.1.0] - 2026-11-11

### Added
- Project Dam: the GoldenEye 007 PC port as a Nintendo Switch homebrew (NRO), started by title takeover from the homebrew menu
- The release archive carries prepare-assets, which converts your own US (NTSC-U) .z64 ROM on a PC; no game content is included
- Renders at 1920x1080 docked and 1280x720 handheld, and follows the console when it is docked or undocked
- The buttons follow the labels printed on the controllers (A confirms, B goes back)
- Aim-mode sensitivity and precision curve for the right stick (Options), set for the short Joy-Con stick: fine aim for headshots, and a trigger press no longer moves the crosshair
- Quit Game in the options menu returns to the HOME menu
- After a crash, data/ge007-crash.log on the SD card records where it happened

### Changed
- Options that have no meaning on the console are hidden: window mode, fullscreen, resolution, MSAA, Restart Game and the update check

### Fixed
- No more stutter on zone changes: textures are no longer rebuilt every time the game reloads them
