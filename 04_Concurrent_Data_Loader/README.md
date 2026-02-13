# Concurrent Data Loader

This project simulates multiple concurrent ingestion processes writing to the same dataset.

## Problem
Parallel ingestion causes:
- Locks
- Deadlocks
- Duplicate records

## Solution
- Write into a staging table
- Use batch IDs
- Merge atomically into final tables

## Enterprise Pattern
This is how companies load data safely at scale.
