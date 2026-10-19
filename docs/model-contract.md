# Model contract

The runnable dashboard backend is `dashboard/backend/app.py` (FastAPI) with inference in `dashboard/backend/pipeline.py`. Both are part of this repository. Empty historical algorithm placeholders have been removed.

## Supported dashboard package

| Artifact | Meaning |
| --- | --- |
| `lstm_leak_classifier.keras` | Sequential LSTM detector with one sigmoid output |
| `lstm_leak_location.keras` | Sequential LSTM locator with seven or eight softmax outputs |
| `scaler.pkl` | Matching fitted StandardScaler, exported with uncompressed joblib |
| `config.pkl` | Dictionary with ordered `feature_cols`, integer `time_steps`, and integer `stride` |
| `manifest.json` | Optional on first import; included on export with sampling interval, class order, alarm policy, and version metadata |

The input shape is `(batch, time_steps, number_of_features)` for both networks. Seven-class packages contain `LOCA`, `LOCAC`, `SGATR`, `SGBTR`, `SLBIC`, `FLB`, and `LLB` in their verified training order. Eight-class packages also include `No leak`. Output width alone cannot establish the correct class order.

CSV inputs need a unique header row, numeric finite `TIME` and configured sensors, strictly increasing timestamps at the declared sampling interval, and at least one complete window. Limits are 25 MB and 50,000 rows per recording. The model package limit is 256 MB.

The importer supports the reference Sequential LSTM/Dense/Dropout architecture, including model filename suffixes such as `_v2`. It intentionally rejects unrelated architectures and custom layers. A compatible shape does not prove that files were trained together; obtain all artifacts from the same run.

For complete import instructions, see the [dashboard guide](../dashboard/README.md#import-the-teams-trained-models). For V3 timing, voting, evaluation differences, and known notebook execution problems, see the [compatibility review](../dashboard/docs/v3-compatibility.md).

## Separate binary detector workflow

A separate experimental workflow, retained outside this dashboard repository, exports `leak_classifier.keras`, `config.json`, and `scaler.pkl`. It defaults to a causal CNN with sensor deltas and does not export a location model.

That format cannot be imported into this dashboard by renaming files. Supporting it requires a separate backend adapter for its features, alarm settings, timestamps, and detector-only result semantics. The current dashboard remains paired with the V3-compatible LSTM service; the standalone workflow is maintained separately.
