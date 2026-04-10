# Mini-UnionFS Simple Test Guide

- `~/unionfs-run`

## 1. One-Time Setup

Run in Terminal 1:

```bash
cd ~/unionfs-run
mkdir -p lower upper mnt
```

(Optional clean reset)

```bash
rm -rf lower upper mnt
mkdir -p lower upper mnt
```

## 2. Prepare Initial Files

Run in Terminal 1:

```bash
cd ~/unionfs-run

echo "this is file1 from lower" > lower/file1.txt
echo "this is file2 from lower" > lower/file2.txt
echo "lower version" > lower/common.txt
echo "upper version" > upper/common.txt
mkdir -p lower/docs
echo "nested lower file" > lower/docs/note.txt
```

## 3. Start UnionFS (Keep This Terminal Running)

Run in Terminal 1:

```bash
cd ~/unionfs-run
./mini_unionfs lower upper mnt
```

If this command is successful, do not close this terminal.

## 4. Run Tests in Terminal 2

Open a second terminal and run:

```bash
cd ~/unionfs-run
```

### Test A: Merged View

```bash
ls -la mnt
```

Expected:
- You should see `file1.txt`, `file2.txt`, `common.txt`, and `docs`.

### Test B: Upper Overrides Lower

```bash
cat mnt/common.txt
```

Expected:
- Output should be `upper version`.

### Test C: Read Lower File Through Mount

```bash
cat mnt/file1.txt
```

Expected:
- Output should be `this is file1 from lower`.

### Test D: Copy-on-Write (CoW)

Modify a lower file through mount:

```bash
echo "new line from mount" >> mnt/file1.txt
```

Now compare lower vs upper:

```bash
echo "--- lower/file1.txt ---"
cat lower/file1.txt
echo "--- upper/file1.txt ---"
cat upper/file1.txt
```

Expected:
- `lower/file1.txt` should NOT have the new line.
- `upper/file1.txt` should have the new line.

### Test E: Create New File Through Mount

```bash
echo "created from mount" > mnt/file3.txt
ls -la upper
```

Expected:
- `file3.txt` should appear in `upper`.
- It should not exist in `lower`.

### Test F: Delete Lower File (Whiteout)

```bash
rm mnt/file2.txt
ls -la upper
ls -la mnt
```

Expected:
- `upper` should contain `.wh.file2.txt`.
- `file2.txt` should disappear from `mnt`.
- `lower/file2.txt` should still physically exist.

Check lower still has file:

```bash
ls -la lower
```

### Test G: Nested Directory Access

```bash
cat mnt/docs/note.txt
```

Expected:
- Output should be `nested lower file`.

### Test H: Directory Creation Through Mount

```bash
mkdir mnt/newdir
ls -la upper
```

Expected:
- `newdir` should be created in `upper`.

### Test I: Rename Lower File

```bash
echo "rename source" > lower/file4.txt
mv mnt/file4.txt mnt/file4_renamed.txt
ls -la upper
cat mnt/file4_renamed.txt
```

Expected:
- `file4_renamed.txt` should be visible in `mnt`.
- A whiteout like `.wh.file4.txt` should appear in `upper`.

### Test J: Truncate File

```bash
echo "123456789" > mnt/file5.txt
truncate -s 4 mnt/file5.txt
cat mnt/file5.txt
```

Expected:
- Output should be `1234`.

### Test K: Remove Directory (rmdir + Whiteout)

```bash
rm -r mnt/docs
ls -la upper | grep .wh.
ls -la mnt
```

Expected:
- `upper` should contain `.wh.docs`.
- `docs` should disappear from `mnt`.
- `lower/docs` should still physically exist.

### Test L: chmod and access behavior

```bash
echo "perm test" > mnt/file6.txt
chmod 000 mnt/file6.txt
ls -l upper/file6.txt
cat mnt/file6.txt
```

Expected:
- Mode should become `----------` (or equivalent 000 mode display).
- `cat` should fail with permission denied for non-root.

Restore permissions:

```bash
chmod 644 mnt/file6.txt
cat mnt/file6.txt
```

Expected:
- File should be readable again.

### Test M: utimens (timestamp update)

```bash
touch -t 202401010101 mnt/file6.txt
stat mnt/file6.txt
stat upper/file6.txt
```

Expected:
- Modified timestamp should be updated.
- Timestamp update should be reflected on the upper copy.

### Test N: chown (optional, needs sudo)

```bash
sudo chown $USER:$USER mnt/file6.txt
ls -ln upper/file6.txt
```

Expected:
- Owner/group values should update on the upper file.
- If running without sudo privileges, you may see an operation not permitted error.

### Test O: getattr-style checks

```bash
stat mnt/file1.txt
stat mnt/file3.txt
```

Expected:
- Both stats should return successfully.
- This confirms metadata resolution from lower/upper as expected.

## 5. Quick Final State Check

```bash
echo "=== lower ==="
find lower -maxdepth 3 -print | sort
echo "=== upper ==="
find upper -maxdepth 3 -print | sort
echo "=== mnt ==="
find mnt -maxdepth 3 -print | sort
```

## 6. Unmount When Done

Run from any terminal:

```bash
fusermount -u ~/unionfs-run/mnt
```

If unmount fails once, try again after closing the terminal that is inside `mnt`.

## 7. Common Mistakes

- Running `./mini_unionfs lower upper mnt` from the wrong directory.
  - Fix: `cd ~/unionfs-run` first.
- Trying to mount on Windows path (`/mnt/d/.../mnt`).
  - Fix: Use Linux path under home like `~/unionfs-run/mnt`.
- Closing the terminal where `mini_unionfs` is running.
  - Fix: Keep that terminal open, use another terminal for tests.
