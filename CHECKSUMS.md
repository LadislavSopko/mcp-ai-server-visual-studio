# VSIX Checksums

SHA-256 of every VSIX published to the [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=LadislavSopko.mcpserverforvs).
The Marketplace is the only distribution channel: any VSIX obtained elsewhere is not ours.

Each hash is taken from the build artifact and verified against the file downloaded back from the Marketplace
(byte-identical) before it is listed here.

| Version | Published | Size (bytes) | SHA-256 |
|---------|-----------|--------------|---------|
| 2.8.5 | 2026-10-27 | 3793213 | `bc2d188e11a2edf835b7ef0a022b24d568892bc1bbdf67e778dbab59f9b678aa` |
| 2.8.4 | 2026-10-26 | 3799579 | `4d1f932b2381cd166362beea044c5ee535b7d0a109f26b83b8f5b661d22dfde2` |
| 2.8.3 | — | 3797948 | `89138222a327ec801a4b026add11727bbdbbf45a05d484826ad79e9bce88d705` |

## What this proves

A matching hash proves that the file you installed is the file we published. It does **not** prove which source
code it was built from — that is a separate claim (see issue #13).

Releases are currently **not code-signed**. Authenticode signing for 0ics srl is in progress; signed releases
will be noted in this table.

## How to verify

Download the exact version from the Marketplace gallery API. The response is **gzip-compressed** — decompress it
before hashing, otherwise the hash will not match.

```bash
VER=2.8.5
curl -sL -o vsix.gz "https://marketplace.visualstudio.com/_apis/public/gallery/publishers/LadislavSopko/vsextensions/mcpserverforvs/$VER/vspackage"
gzip -dc vsix.gz > mcpserverforvs-$VER.vsix
sha256sum mcpserverforvs-$VER.vsix
```

PowerShell 7+ (Windows):

```powershell
$ver = "2.8.5"
Invoke-WebRequest "https://marketplace.visualstudio.com/_apis/public/gallery/publishers/LadislavSopko/vsextensions/mcpserverforvs/$ver/vspackage" -OutFile "mcpserverforvs-$ver.vsix"
Get-FileHash "mcpserverforvs-$ver.vsix" -Algorithm SHA256
```

(PowerShell 7's `Invoke-WebRequest` decompresses the gzip response automatically — verified with 7.6.4.)
