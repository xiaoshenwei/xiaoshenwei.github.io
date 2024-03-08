---
toc: true
title: "python常用小工具"
categories:
  - developer
tags:
  - python
---

# 文件按行分块读取

```python
import itertools
import tempfile

class FileReader:
    def __init__(self, file_path: str):
        self.file_path = file_path
        self.file = None

    def __enter__(self):
        self.file = open(self.file_path, "r")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.file:
            self.file.close()

    def chunk(self, chunk_size: int = 20):
        while True:
            buffer = list(itertools.islice(self.file, chunk_size))
            if not buffer:
                break
            yield buffer


if __name__ == "__main__":
    file = tempfile.NamedTemporaryFile(delete=False)
    for i in range(10):
        file.write(f"Test {i}\n".encode())
    file.close()
    with FileReader(file.name) as reader:
        for chunk in reader.chunk(3):
            print(chunk)
    print("Done!")

```