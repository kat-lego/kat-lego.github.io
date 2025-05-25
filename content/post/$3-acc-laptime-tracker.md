+++
author = "katlego modupi"
title = "acc laptime tracker"
date = "2025-06-01"
description = "building a program to track my laptimes in assetto corsa competizione"
draft = true
tags = [
    "go", "acc", "firebase"
]
+++

Building a program to track my lap times in assetto corsa competizione.

[![github repo](https://img.shields.io/badge/acc_laptime_tracker-gray?logo=github)](https://github.com/kat-lego/acc-laptime-tracker)
<!--more-->

Some time last year, I decided to invest money on Assetto Corsa Competizione and a racing wheel. I
have had a hard time investing the time it takes to 'git gud'. Now is the time I feed into my
delusions of grandeur and top the lap times. But first, I will need a bespoke lap time tracker.

## why ?

Glory, Fame and Prestige.

Or at the very least, I’d like to feel like I’m getting my money’s worth out of my racing wheel setup. Tracking lap times isn’t just about stats — it’s a tool for improvement. Seeing my performance trends across different tracks, cars, or setups will help me understand what’s working and what’s not.

## requirements

At a high level, I need the following:
* Read telemetry data from ACC: The game exposes data through a shared memory interface. I'll use that to read things like current lap time, session state, and car info.
* Detect new sessions, laps and lap sectors
* Persist the data: I want to store lap times (and maybe car/track/session metadata) in the cloud — Firebase is a good fit for quick prototyping.
* Show progress: present the metrics on a web application

## the stack

In this stack, I will be using golang to build the program. No justification will be given at this
time.

### live data
As mentioned ACC exposes its game data through a shared memory interface. The interface consists of
3 memory mapped files it creates.

#### memory mapped file
A memory-mapped file is a mechanism that maps the contents of a file (or a portion of it) directly
into a process's virtual memory space. This allows the application to read from and write to the
file as if it were accessing regular memory (RAM), rather than using traditional file I/O functions
like `read()` or `write()`.

This is how it works:

* Given the file you want to map

```
+-----------------------+
|    File: data.bin     |
+-----------------------+
| Page 0 | Page 1 | ... |
+-----------------------+
```

* The OS maps the file (or a portion) into the process’s virtual memory:

```
Process Virtual Address Space
+---------------------------------------------+
|   ...   | 0x1000 | 0x1001 | ... | 0x1FFF    |
+---------------------------------------------+
             ▲
             │
    File content appears here (read/write like an array)
```

* The virtual address space is backed by physical RAM. 

```
+-------------------------+     +-------------------------+
|   Virtual Address:      |     |  Physical RAM Page:     |
|     0x1000 → 0x1FFF     | --> |  Loaded from disk page  |
|                         |     |                         |
+-------------------------+     +-------------------------+
```

* If you access a region that hasn’t been loaded yet, the OS will:
    * Trigger a page fault
    * Load the required part of the file into RAM
    * Update the page table to reflect this mapping

* ACC uses a non persistent memory file, which means the file only exists loaded on to physical RAM.

#### api overview
The Assetto Corsa Competizione shared memory interface provides 3 memory files, being:

* **`physics`**: contains data about the car's physics state (e.g., suspension, tire slip, speed, G-forces).
* **`graphics`**: focuses on session-level data and what is rendered on screen, such as lap times, positions, and session state.
* **`static`**: holds information that doesn't change during a session, like car model, track name, and max values for power or fuel.

You can find the full documentation on this blog post [ACC Shared Memory Documentation](https://www.assettocorsa.net/forum/index.php?threads/acc-shared-memory-documentation.59965/).

#### reading from the file
This is how we will be reading the data from ACC's shared memory.

1. We will define the following function that will read from the named file and return an instance
   of type T
```go
func ReadSharedMemoryStruct[T any](name string) (*T, error)
```
2. Load windows kernel dll to get access to `OpenFileMappingW` and `MapViewOfFile` from the
   windows api

3. Use Open the shared memory file exposed by the game (via Windows' `CreateFileMappingW` )
4. Map the data to the go process (via Windows `MapViewOfFile`)
5. Cast the raw byte slice to a struct using `unsafe`.

All in all the program looks like so
```go
func ReadSharedMemoryStruct[T any](name string) (*T, error) {
	namePtr, err := syscall.UTF16PtrFromString(name)
	if err != nil {
		return nil, fmt.Errorf("failed to encode name: %w\n", err)
	}

	hMap, _, callErr := procOpenFileMapping.Call(
		FILE_MAP_READ,
		0,
		uintptr(unsafe.Pointer(namePtr)),
	)
	if hMap == 0 {
		return nil, fmt.Errorf("OpenFileMappingW failed: %w\n", callErr)
	}
	defer procCloseHandle.Call(hMap)

	structSize := unsafe.Sizeof(new(T))
	addr, _, callErr := procMapViewOfFile.Call(
		hMap,
		FILE_MAP_READ,
		0,
		0,
		uintptr(structSize),
	)
	if addr == 0 {
		return nil, fmt.Errorf("MapViewOfFile failed: %w\n", callErr)
	}
	defer procUnmapViewOfFile.Call(addr)

	value := *(*T)(unsafe.Pointer(addr))
	return &value, nil
}
```

### tracking sessions
To track the session data, the program reads the memory mapped files from the graphics, and static
info interfaces. We aggregate all the information into one struct that looks as follows.

```go
type AccGameState struct {
	SharedMemoryVersion string `json:"sharedMemoryVersion"`
	AssettoCorsaVersion string `json:"assettoCorsaVersion"`

	Status       string  `json:"status"`
	SessionType  string  `json:"sessionType"`
	Track        string  `json:"track"`
	CarModel     string  `json:"carModel"`
	SectorCount  int32   `json:"sectorCount"`
	NumberOfCars int32   `json:"numberOfCars"`
	Clock        float32 `json:"clock"`

	CompletedLaps   int32 `json:"completedLaps"`
	BestLapTime     int32 `json:"bestLapTime"`
	PreviousLapTime int32 `json:"previousLapTime"`

	CurrentLapTime     int32 `json:"currentLapTime"`
	CurrentSectorIndex int32 `json:"sectorIndex"`
	PreviousSectorTime int32 `json:"previousSectorTime"`
	IsValid            bool  `json:"isValid"`
	IsInPitLane        bool  `json:"isInPitLane"`
	IsInPit            bool  `json:"isInPit"`
}
```

I have implemented a service layer in go that maintains details of the current session. The current
session, contains a list of laps and each lap has a list of sectors. This is how the session is
modelled.

```go
type Session struct {
	Id              string `json:"id"`
	StartTime       int64  `json:"startTime"`
	SessionType     string `json:"sessionType"`
	Track           string `json:"track"`
	CarModel        string `json:"carModel"`
	NumberOfSectors int32  `json:"numberOfSectors"`
	CompletedLaps   int32  `json:"lapsCompleted"`
	BestLap         int32  `json:"bestLap"`
	IsActive        bool   `json:"isActive"`
	Player          string `json:"player"`

	Laps []*Lap `json:"laps"`
}

type Lap struct {
	LapNumber int32 `json:"lapNumber"`
	LapTime   int32 `json:"lapTime"`
	LapDelta  int32 `json:"lapDelta"`
	IsValid   bool  `json:"isValid"`
	IsActive  bool  `json:"isActive"`

	LapSectors []*LapSector `json:"lapSectors"`
}

type LapSector struct {
	SectorNumber int32 `json:"sectorNumber"`
	SectorTime   int32 `json:"sectorTime"`
	IsActive     bool  `json:"isActive"`
}

```

In this we track the latest
lap, and sector. I use the following logic to check if a new session, lap or sector has
started or ended:

**`session`**
* if the `CompletedLaps` from the new state data is less than the number of completed laps tracked in
  the current session, then we start a new session (and complete the old one)
* To complete the old session involves setting the session to `InActive` and tries to complete the
  last lap (refer to the next session on what completing a lap entails)
* `NOTE`: one of the current shortfalls of this, is when I restart a session before completing a lap.
  (however this is not that bad)

**`lap`**
* if the `CompletedLaps` is greater that the number of completed laps we are tracking the session,
then we start a new lap (and complete the old one)
* to complete the old lap, we set the lap time to the `PreviousLapTime` from the `AccGameState`. We
  also set the sector time to `PreviousLapTime`

**`sector`**
* if the `CurrentSectorIndex` is different to the sector number we are tracking, then we start a 
new sector (and complete the old one)
* to complete the old sector, we set the sector time to the `PreviousSectorTime` from the 
`AccGameState`.

Any time we complete and create or update session info, we update/write it to the database.

### saving
To persist the sessions, I decided on Google's Firestore database. This is a NoSql database which
has a decent free allocation and should work for my small loads. No further thought was needed on this. The approach I took was to lazily save the same schema presented for the session.

### api
To make the lap data publicly accessible, I added an API hosted on hosted on google cloud run. The
API is running on a docker container and was developed in go, using go gin as the web framework that
handles the requests. You can check it out at `TODO: add webshield image with link`. All the API
does is expose a GET endpoint which retrieves the 20 most recent session data and no more.

#### protecting the database
TODO

#### protecting the api
TODO

### web ui
As a way to present the data, I have build a static website. In service of furthering my golang
theme, I have decided to choose the wrong tool for the job, Hugo.

### shell ui (bonus)

And this completes the stack.
