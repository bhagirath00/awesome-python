# Parsing Log Files: Memory-Efficient File Traversals

Log files are simple to write, but to get value from them, you must be able to read and analyze their contents. Whether you are searching for malicious IP addresses scanning your server, counting how many error codes occurred yesterday, or compiling average response times, Python is an excellent tool for log parsing.

However, in production environments, log files can grow to several gigabytes. Loading an entire 5GB file into memory using `file.read()` or `file.readlines()` will exhaust your server's RAM, crash the execution container, and impact other system services.

In this guide, we will examine the memory constraints of file reading, implement the **Generator Pattern** to read files line-by-line with a minimal memory footprint, and parse structured log entries using standard string manipulation techniques.

---

## Memory Footprints: Read vs. Iterator

If you use `readlines()`, Python loads every line of the file into RAM as a list of strings:

```mermaid
graph TD
    subgraph ReadAll [file.readlines - Memory Heavy]
        File1[5GB Log File] -->|Load entire file| RAM[5GB RAM consumed]
        RAM --> Process1[Process List of Strings]
    end
    
    subgraph Iterator [line-by-line Iterator - Memory Efficient]
        File2[5GB Log File] -->|Stream line 1| RAM2[1KB RAM consumed]
        RAM2 --> Process2[Process Line 1]
        File2 -->|Stream line 2| RAM2
        RAM2 --> Process2
    end
```

Using Python's **file iterator protocol** allows you to stream files line-by-line, keeping your memory usage constant regardless of whether the file is 5MB or 50GB.

---

## 1. Memory-Efficient Reading (The Generator Pattern)

By using Python's `yield` statement, we can write a generator function that returns one line at a time. The file remains open, and the generator pauses execution between lines, consuming almost no memory.

```python
import os
import sys

def stream_log_lines(filepath):
    """
    Generator that streams log lines one-by-one.
    """
    if not os.path.exists(filepath):
        print(f"Error: Log file not found at '{filepath}'")
        return

    try:
        # We use standard context manager to safely open and close the file
        with open(filepath, "r", encoding="utf-8") as f:
            for line in f:
                # Yield passes the line back to the caller and pauses execution
                yield line
    except Exception as e:
        print(f"Failed to read file: {e}")
```

---

## 2. Basic Parsing and Tokenization

If your log file has a consistent, space-separated or comma-separated layout, you can parse it using Python's string manipulation methods like `.split()`, `.strip()`, and `.partition()`.

Consider this standard log line structure:
`[TIMESTAMP] [LEVEL] [COMPONENT] MESSAGE`

```python
# Sample log line
sample_line = "[2026-06-27 18:05:00] [ERROR] [database] Connection timeout occurred."

def parse_log_line(line):
    line = line.strip()
    if not line:
        return None

    try:
        # Split using bracket boundaries
        # We limit the split count using maxsplit to prevent split errors in the message content
        parts = line.split("] ")
        
        timestamp = parts[0].replace("[", "")
        level = parts[1].replace("[", "")
        component = parts[2].replace("[", "")
        message = parts[3]
        
        return {
            "timestamp": timestamp,
            "level": level,
            "component": component,
            "message": message
        }
    except IndexError:
        # Line does not conform to the expected format
        return None

print(parse_log_line(sample_line))
```

---

## 3. High-Performance Log Level Counter

Let's combine our memory-efficient generator and the string parser to compile statistics on a log file:

```python
from collections import Counter
import psutil
import os

def parse_and_compile_stats(log_filepath):
    level_counter = Counter()
    processed_lines = 0
    
    # Track initial memory usage
    process = psutil.Process(os.getpid())
    initial_memory = process.memory_info().rss / (1024 * 1024)
    print(f"Initial Memory Footprint: {initial_memory:.2f} MB")
    
    # Process line-by-line
    for line in stream_log_lines(log_filepath):
        parsed = parse_log_line(line)
        if parsed:
            level_counter[parsed["level"]] += 1
            processed_lines += 1
            
        # Log memory metrics every 100,000 lines
        if processed_lines % 100000 == 0:
            current_memory = process.memory_info().rss / (1024 * 1024)
            print(f"Lines parsed: {processed_lines} | Current Memory: {current_memory:.2f} MB")

    # Print results
    print("\n==================================================")
    print("📋 LOG PARSING SUMMARY")
    print("==================================================")
    print(f"Total Lines Parsed: {processed_lines}")
    print("\nLog Level Distribution:")
    for level, count in level_counter.items():
        print(f" - {level}: {count}")
    
    final_memory = process.memory_info().rss / (1024 * 1024)
    print(f"\nFinal Memory Footprint: {final_memory:.2f} MB")
    print("==================================================")
```

---

## Summary

Reading log files line-by-line using generators is a vital pattern for analyzing large files without exhausting system resources. In the next section, we will cover **Log Aggregation** concepts.
