Fork
====
This fork is for analysing problems under Windows 10 with gpg encryption in msys2 / mingw environment.

How to Build a Windows version for debugging
--------------------------------------------
For local build the description from here works well: 
https://stackoverflow.com/questions/43040370/how-to-install-git-crypt-on-windows



    1. Install msys2:

    http://www.msys2.org/

    2. Open a msys2 terminal. Then ...

    3. Install g++ for windows, following the instructions here:

    https://github.com/orlp/dev-on-windows/wiki/Installing-GCC--&-MSYS2

    4. Be sure /mingw64/bin is in your path. (e.g. which g++)

    5. git clone git@github.com:AGWA/git-crypt

    6. cd git-crypt

    7. make
    
To build a version runnable without the libraries "libstdc++-6.dll" and "libcrypto-1_1-x64.dll" in step 7 the libraries can be linked static:

    7. make LDFLAGS="-static-libstdc++ -static -lcrypto.dll"

Here is the output of the make file

```
> make LDFLAGS="-static-libstdc++ -static -lcrypto.dll"
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o git-crypt.o git-crypt.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o commands.o commands.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o crypto.o crypto.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o gpg.o gpg.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o key.o key.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o util.o util.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o parse_options.o parse_options.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o coprocess.o coprocess.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o fhstream.o fhstream.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o crypto-openssl-10.o crypto-openssl-10.cpp
g++ -c  -Wall -pedantic -Wno-long-long -O2 -std=c++11  -o crypto-openssl-11.o crypto-openssl-11.cpp
g++ -Wall -pedantic -Wno-long-long -O2 -std=c++11 -o git-crypt git-crypt.o commands.o crypto.o gpg.o key.o util.o parse_options.o coprocess.o fhstream.o crypto-openssl-10.o crypto-openssl-11.o -static-libstdc++ -static -lcrypto.dll
```


Analysis
-------
mkdir_parents fails, because GetFileAttributes() returns for every file "/c/..." that it is a directory (16), even if it does not exist.

stat instead of GetFileAttributes is not better.

https://www.msys2.org/wiki/Porting/

Path-Converter sources:
https://github.com/msys2/path_convert/blob/master/src/path_conv.h




git-crypt - transparent file encryption in git
==============================================

git-crypt enables transparent encryption and decryption of files in a
git repository.  Files which you choose to protect are encrypted when
committed, and decrypted when checked out.  git-crypt lets you freely
share a repository containing a mix of public and private content.
git-crypt gracefully degrades, so developers without the secret key can
still clone and commit to a repository with encrypted files.  This lets
you store your secret material (such as keys or passwords) in the same
repository as your code, without requiring you to lock down your entire
repository.

git-crypt was written by [Andrew Ayer](https://www.agwa.name) (agwa@andrewayer.name).
For more information, see <https://www.agwa.name/projects/git-crypt>.

Building git-crypt
------------------
See the [INSTALL.md](INSTALL.md) file.


Using git-crypt
---------------

Configure a repository to use git-crypt:

    cd repo
    git-crypt init

Specify files to encrypt by creating a .gitattributes file:

    secretfile filter=git-crypt diff=git-crypt
    *.key filter=git-crypt diff=git-crypt
    secretdir/** filter=git-crypt diff=git-crypt

Like a .gitignore file, it can match wildcards and should be checked into
the repository.  See below for more information about .gitattributes.
Make sure you don't accidentally encrypt the .gitattributes file itself
(or other git files like .gitignore or .gitmodules).  Make sure your
.gitattributes rules are in place *before* you add sensitive files, or
those files won't be encrypted!

Share the repository with others (or with yourself) using GPG:

    git-crypt add-gpg-user USER_ID

`USER_ID` can be a key ID, a full fingerprint, an email address, or
anything else that uniquely identifies a public key to GPG (see "HOW TO
SPECIFY A USER ID" in the gpg man page).  Note: `git-crypt add-gpg-user`
will add and commit a GPG-encrypted key file in the .git-crypt directory
of the root of your repository.

Alternatively, you can export a symmetric secret key, which you must
securely convey to collaborators (GPG is not required, and no files
are added to your repository):

    git-crypt export-key /path/to/key

After cloning a repository with encrypted files, unlock with GPG:

    git-crypt unlock

Or with a symmetric key:

    git-crypt unlock /path/to/key

That's all you need to do - after git-crypt is set up (either with
`git-crypt init` or `git-crypt unlock`), you can use git normally -
encryption and decryption happen transparently.

Current Status
--------------

The latest version of git-crypt is [0.6.0](NEWS.md), released on
2017-11-26.  git-crypt aims to be bug-free and reliable, meaning it
shouldn't crash, malfunction, or expose your confidential data.
However, it has not yet reached maturity, meaning it is not as
documented, featureful, or easy-to-use as it should be.  Additionally,
there may be backwards-incompatible changes introduced before version
1.0.

Security
--------

git-crypt is more secure than other transparent git encryption systems.
git-crypt encrypts files using AES-256 in CTR mode with a synthetic IV
derived from the SHA-1 HMAC of the file.  This mode of operation is
provably semantically secure under deterministic chosen-plaintext attack.
That means that although the encryption is deterministic (which is
required so git can distinguish when a file has and hasn't changed),
it leaks no information beyond whether two files are identical or not.
Other proposals for transparent git encryption use ECB or CBC with a
fixed IV.  These systems are not semantically secure and leak information.

Limitations
-----------

git-crypt relies on git filters, which were not designed with encryption
in mind.  As such, git-crypt is not the best tool for encrypting most or
all of the files in a repository. Where git-crypt really shines is where
most of your repository is public, but you have a few files (perhaps
private keys named *.key, or a file with API credentials) which you
need to encrypt.  For encrypting an entire repository, consider using a
system like [git-remote-gcrypt](https://spwhitton.name/tech/code/git-remote-gcrypt/)
instead.  (Note: no endorsement is made of git-remote-gcrypt's security.)

git-crypt does not encrypt file names, commit messages, symlink targets,
gitlinks, or other metadata.

git-crypt does not hide when a file does or doesn't change, the length
of a file, or the fact that two files are identical (see "Security"
section above).

git-crypt does not support revoking access to an encrypted repository
which was previously granted. This applies to both multi-user GPG
mode (there's no del-gpg-user command to complement add-gpg-user)
and also symmetric key mode (there's no support for rotating the key).
This is because it is an inherently complex problem in the context
of historical data. For example, even if a key was rotated at one
point in history, a user having the previous key can still access
previous repository history. This problem is discussed in more detail in
<https://github.com/AGWA/git-crypt/issues/47>.

Files encrypted with git-crypt are not compressible.  Even the smallest
change to an encrypted file requires git to store the entire changed file,
instead of just a delta.

Although git-crypt protects individual file contents with a SHA-1
HMAC, git-crypt cannot be used securely unless the entire repository is
protected against tampering (an attacker who can mutate your repository
can alter your .gitattributes file to disable encryption).  If necessary,
use git features such as signed tags instead of relying solely on
git-crypt for integrity.

Files encrypted with git-crypt cannot be patched with git-apply, unless
the patch itself is encrypted.  To generate an encrypted patch, use `git
diff --no-textconv --binary`.  Alternatively, you can apply a plaintext
patch outside of git using the patch command.

git-crypt does not work reliably with some third-party git GUIs, such
as [Atlassian SourceTree](https://jira.atlassian.com/browse/SRCTREE-2511)
and GitHub for Mac.  Files might be left in an unencrypted state.

Gitattributes File
------------------

The .gitattributes file is documented in the gitattributes(5) man page.
The file pattern format is the same as the one used by .gitignore,
as documented in the gitignore(5) man page, with the exception that
specifying merely a directory (e.g. `/dir/`) is *not* sufficient to
encrypt all files beneath it.

Also note that the pattern `dir/*` does not match files under
sub-directories of dir/.  To encrypt an entire sub-tree dir/, use `dir/**`:

    dir/** filter=git-crypt diff=git-crypt

The .gitattributes file must not be encrypted, so make sure wildcards don't
match it accidentally.  If necessary, you can exclude .gitattributes from
encryption like this:

    .gitattributes !filter !diff
