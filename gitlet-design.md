# Gitlet Design Document

**Name**: Akshay Thakur

## Miscellaneous:
- Don’t use a HashMap in a class that is going to be serialized, use a TreeMap instead

- HashMap fine if it’s being serialized outside of a class? Just the object itself being written to a file?

- Rm-branch: need to be able to delete reference to a branch in constant time, HashMap worst case seems to be linear

- Store branches as objects/files instead, don’t use a data structure

- Need to go back through commands where user inputs a commit ID (checkout, reset): command should function the same if shortened ID input (anywhere from 1-40 characters), string methods may be helpful here, don’t have to worry about ambiguity of shortened ID

   - Implementation: use substring and equals, comparing shortened ID against list of full IDs from input list of IDs (method called getFullCommitID in Gitlet.java)

## Directories:
Working directory:

- .gitlet

- branches: (each branch is one file)

- commits: (each commit is one file, named using string ID)

- staging: (each blob in staging area is one file, named using string ID)

- blobs: (each blob is one file, named using string ID)

#### Gitlet object, contains following objects within:

- CommitTree: (maps names of commits)

- StagingArea

- StagingAreaRemoval

- CurrentBranch

- CurrentHead

## Classes:
### Main
Check if .gitlet directory exists
- If yes: return error message
- If no: proceed with initialization
For all commands except init, check if .gitlet exists:
- If yes: proceed
- If no: return error message
Failure cases:
- If a user doesn't input any arguments, print the message Please enter a command. and exit.
- If a user inputs a command that doesn't exist, print the message No command with that name exists. and exit.
- If a user inputs a command with the wrong number or format of operands, print the message Incorrect operands. and exit.
- If a user inputs a command that requires being in an initialized Gitlet working directory (i.e., one containing a .gitlet subdirectory), but is not in such a directory, print the message Not in an initialized Gitlet directory.

### Commit
- Initialization (see object description below)
- Equals method (should override default, compare string IDs)

### Gitlet class
- Should hold all loose objects to make deserialization/serialization easier
Contains:
- CommitTree: (maps names of commits)
- StagingArea
- StagingAreaRemoval
- CurrentBranch
- CurrentHead

### Branch
Each branch is one branch object

Contains:
- Name of branch
- BranchHead: name of this branch’s head commit
- Serialized as file in the branches subdirectory, filename is name of branch

## Objects:

### Commit tree (TreeMap):
Links references to commits (only uses integer ID of commits, not actual objects?)

Object serialized and stored in gitlet directory for persistence

### Commits:
- Log message (given)
- Timestamp: use java.util.Date and java.util.Formatter
- Author?
- Reference to parent commit
- Second parent commit for merging?
- Reference to blobs (HashMap TreeMap?)
  - Key: filename
  - Value: string, unique universal IDs given by SHA-1 code
- Universal ID
- Once commit object created, serialize the object, use that as input to SHA-1 method, produces unique integer ID
  - Integer ID is what is tracked by the commit tree, not the actual object itself

### Staging Area (TreeMap):
- Key: file name (from working directory)
- Value: SHA-1 hash

### Staging Area Removal Files (ArrayList):
- Files in the staging area that are marked for removal

### CurrentBranch (name of branch)

### CurrentHead (commit ID)

## Commands:

### Init:
- Check if .gitlet exists:
  - If yes: print the error message A Gitlet version-control system already exists in the current directory.
- Create new commit tree
  - One universal commit: The timestamp for this initial commit will be 00:00:00 UTC, Thursday, 1 January 1970 in whatever format you choose for dates (this is called   - "The (Unix) Epoch", represented internally by the time 0.)
- Create .gitlet internal directory structure

### Add:
- Takes input file name, looks for it in working directory
- If file doesn’t exist in the working directory: print the error message File does not exist. and exit without changing anything
- File fed into SHA-1 method, produces unique integer ID based on the file’s contents AND filename

If you want it to incorporate the filename, you’ll have to come up with a hashing scheme that does so (e.g. hash(hash(filename) + hash(content)) or something)

- If staging area HashMap has filename (working directory), compare values (SHA-1):
  - If equal: delete reference to file in staging area HashMap, and delete file in staging area directory
  - If not: delete file in staging area directory with current SHA-1 value, replace SHA-1 value in staging area HashMap with new SHA-1, add file with new SHA-1 value to staging area directory
- Copy of file created with readContents/writeContents (read original file, write it to the new location), using the SHA-1 hash as the file name
- Copy added to staging directory
- Name of file/copy (SHA-1) added to staging area HashMap

### Commit:
- If staging area empty, return error message: print the message No changes added to the commit.
- If no commit message passed in: print the error message Please enter a commit message.
- Create new commit object (set parent reference (CommitHead), log message, commit time)
- Copy files CommitHead is referencing (copy contents of previous commit’s blob TreeMap)
- Go through files in staging area TreeMap:
  - If file marked for removal:
    - Delete key and value in commit’s TreeMap
    - If key/filename contained in commit’s blob TreeMap (see if commit contains the same filename):
      - Maybe check to see if SHA-1 is the same, if it is, no need to change the value or add the file (shouldn’t be needed)
      - Change value in TreeMap to the staging area file’s SHA-1/name/value
      - Read contents from staging area directory and write to blobs directory using SHA-1 value
  - If not:
    - Add new key and value to new commit’s TreeMap
    - Read contents from staging area directory and write to blobs directory using SHA-1 value
    - Add new commit object to commit tree
    - Serialize commit object
    - Run serialized object through SHA-1, write serialization to a file in the commits subdirectory with SHA-1 value as the name
    - Lookup current branch, deserialize from the branches subdirectory, change commit ID value of object to the ID of the new commit just added, reserialize branch object
    - Update CommitHead
    - Clear staging area directory AND TreeMap
    - Clear Removal ArrayList

### Rm:
- If filename is a key in the staging area HashMap, remove it from the HashMap and the staging area directory
- If (NOT else if) head commit blob TreeMap has key with filename, add it to the staging area HashMap and add it to the Removal ArrayList
- Also delete the file from the working directory if it still exists (use Utils.restrictedDelete
- Consider a directory dir, within which you have a file hello.txt. If you ran init in this directory, there should be a subdirectory, .gitlet living inside dir. Then, if you called restrictedDelete on hello.txt, it would delete hello.txt because dir contains the directory .gitlet. If you were to call restrictedDelete without having init’d, meaning there was no .gitlet subdirectory, then it will throw an IllegalArgumentException. restrictedDelete will also not delete directories
- Else error message: print the error message No reason to remove the file

### Log:
- Currently not using information from the commitTree itself, may be simpler to use it in some capacity?
- Deserialize head commit
- Set temp to head commit
- While parent commit is not null (last one should be the initial commit)
  - Print “===”
  - Print “commit” + SHA-1 (name of commit)
  - If second parent commit is not null:
    - Print “Merge: “ + first 7 digits of first parent’s commit ID + space + first 7 digits of second parent’s commit ID
    - Print “Date: “ + commit timestamp
    - Print commit log message
  - If not initial commit: print newline
  - Get name of parent commit, deserialize it from the commits subdirectory
  - Set temp to first parent of commit

### Global-log:
- Use Utils.plainFilenamesIn(.gitlet/commits) to get List of all commits
- Iterate over elements in List
- Deserialize current indexed commit from the commits subdirectory
- Print “===”
- Print “commit” + SHA-1 (name of commit)
- If second parent commit is not null:
- Print “Merge: “ + first 7 digits of first parent’s commit ID + space + first 7 digits of second parent’s commit ID
- Print “Date: “ + commit timestamp
- Print commit log message
- If not last commit in the list: print newline

### Find:
- Use Utils.plainFilenamesIn(.gitlet/commits) to get List of all commits
- Initialize counter
- Iterate over elements in List
- Deserialize current indexed commit from the commits subdirectory
- If input commit message equals current commit message:
  - Print commit ID
  - Increment counter
- If counter equals 0:
  - Print error message Found no commit with that message.

### Status:
- Print “=== Branches ===”
- Get list of branch names from branches subdirectory using Utils method
- Iterate over list:
  - If current string equals current branch name, print * + branch name
  - Else: print branch name
- Print newline
- Print “=== Staged Files ===”
- Iterate over files in staging area HashMap:
  - If file name NOT in Removal ArrayList: print filename
- Print newline
- Print “=== Removed Files ===”
- Sort Removal Arraylist, iterate over files, print filename
- Print newline
- Print “=== Modifications Not Staged For Commit ===”
- Print newline
- Print “=== Untracked Files ===”

### Checkout:
-- [file name]

- Deserialize CommitHead
- Find blob ID by using head commit’s blob TreeMap and the input filename
- If filename is not in blob TreeMap, error: printing the error message File does not exist in that commit.
- Read contents of the file from the blobs subdirectory
- Write contents to working directory with the same filename

[commit id] -- [file name]

- Use input commit ID to deserialize commit object from commits subdirectory
- If the commit ID is not in the subdirectory, error: print No commit with that id exists.
- Find blob ID by using commit’s blob TreeMap and the input filename
- If filename is not in blob TreeMap, error: printing the error message File does not exist in that commit.
- Read contents of the file from the blobs subdirectory
- Write contents to working directory with the same filename

[branch name]

- If branch name is not branches subdirectory, error: print No such branch exists.
- If current branch equals input branch name, error: print No need to checkout the current branch.
- Deserialize input branch name, get that branch’s head commit ID)
- Deserialize replacement commit object from commit subdirectory
- Get CommitHead
- Iterate over all blobs in replacement commit’s blob TreeMap (TM1):
    - If filename/key is not contained in current commit’s blob TreeMap AND filename is in CWD: error, print There is an untracked file in the way; delete it, or add and commit it first.
- Iterate over all blobs in replacement commit’s blob TreeMap (TM1):
  - Read contents of the file from the blobs subdirectory
  - Write contents to working directory with the same filename
- Iterate over all blobs in current commit’s blob TreeMap:
  - If filename/key is not contained in TM1, delete file with that filename from the CWD
- Clear staging area HashMap and subdirectory
- Set current branch equal to input branch name
- Set CommitHead to input branch’s head via branch object

### Branch:
- If the input branch name is already in branches subdirectory, print the error message A branch with that name already exists.
- Create new branch object, serialize, add to branches subdirectory:
  - Key: input branch name
  - Value: CurrentHead

### Rm-branch:
- If the input branch name not in branches subdirectory, print the error message A branch with that name does not exist.
- If input branch name equals CurrentBranch, error: printing the error message Cannot remove the current branch.

### Reset:
- If input commit ID (may be shortened) doesn’t match any commits in commits subdirectory, error:  print No commit with that id exists.  
- Deserialize commit with the input commit ID from the commits subdirectory
- Should work with shortened commit ID
- Deserialize CurrentHead commit
- Iterate over files in working directory
  - If filename not in CurrentHead blob TreeMap AND filename is in input commit blob TreeMap, error: print There is an untracked file in the way; delete it, or add and commit it first.  
- Iterate over all files in the HeadCommit’s blob TreeMap
  - If filename not in input commit’s blob TreeMap, delete from CWD
- Iterate over all files in the input commit’s blob TreeMap
  - Read contents of the file from the blobs subdirectory
  - Write contents to working directory with the same filename
- Deserialize current branch, change BranchHead to input commit name (full ID), reserialize
- Change CurrentHead’s value to the full version of the input commit ID
- Clear staging area
