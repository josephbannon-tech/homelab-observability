# MediaCorruptionDetected

| Severity | Condition |
|----------|-----------|
| warning | `media_corrupt_reports_count > 0`, sustained 2m |

**Signal**: the media-integrity pipeline confirmed a downloaded file fails decode verification (two-pass ffmpeg scan with a reproducible error offset). The pipeline has already acted: the torrent is stopped, the file deleted (single-file case) or report-only (a corrupt episode inside a multi-file pack is reported, not deleted, to protect the good episodes). The metric counts report files awaiting operator action, so **the alert is a to-do item, not an incident**.

## Operator loop

1. Read the report file(s): `/srv/media-corrupt-reports/corrupt-*.txt` on JBVM01, which name the download and release group.
2. Re-acquire the item **from a different source/release group** (the same release will fail the same way).
3. Delete the report file(s). The count drops to 0 within about a minute and the alert clears itself.

## Trust boundaries, learned the hard way

- **The detector's CORRUPT verdict is reliable** (validated against deterministic reproduction and visible playback artifacts), and the user's eyes are ground truth over any byte-level spot check: a 16-byte read at the reported offset proving "the bytes look fine" proves nothing about stream structure.
- **UNREADABLE is not CORRUPT.** A file the scanner cannot open (permissions, access) must never be treated as corrupt; that mistake once deleted a clean file. The pipeline classifies unreadable separately and never deletes on it, so if reports mention access errors, fix the access, don't re-acquire.
- **Decoder errors only.** ffmpeg's null-muxer warnings ("non monotonically increasing dts") on remuxes are benign and are filtered; a sudden flood of FLAKY verdicts on large remuxes suggests that filter regressed, not that the library rotted.
