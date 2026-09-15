# Fehleranalyse: `ModuleNotFoundError: No module named 'sympy'`

## Kurzfassung
Der Start bricht mit `ModuleNotFoundError: No module named 'sympy'`. Das ist **kein** echtes Fehlpaket — `sympy` ist im `uv.lock` korrekt enthalten (als Transitiv-Dependency von `networkx`, v1.14.0). Es wurde **bewusst im Docker-Image gelöscht** (Zeile 38–39 `Dockerfile`). Diese Cleanup-Entscheidung kollidiert mit dem `@torch.compiler.disable`-Decorator der installierten `pocket-tts`-Version.

## Betroffene Umgebung
- Image: Docker, Python 3.13 (`/usr/local/lib/python3.13`), CPU-only `torch==2.12.0`
- `pocket-tts` 2.1.0 (aus `uv.lock`)
- `networkx` (→ `sympy`), beide im Cleanup gelöscht

## Import-Kette (Beweis aus dem Trace)
Der Crash ist ein **Import-Zeitpunkt**-Problem, kein Laufzeitproblem. `sympy` wird beim *Laden* der `pocket_tts`-Module benötigt, noch bevor der Server irgendetwas synthetisiert:

```
__main__.py
→ pocket_tts/__init__.py
→ pocket_tts.models.tts_model: TTSModel
→ pocket_tts.models.flow_lm: FlowLMModel
→ pocket_tts.modules.transformer: StreamingTransformer
→ pocket_tts.modules.attention: StreamingMultiheadAttention, _cached_causal_mask
→ attention.py:39   @torch.compiler.disable      ← Module-level Decorator
      └─> torch/compiler/__init__.py:256 disable()
           └─> import torch._dynamo
                └─> torch/_dynamo/convert_frame.py:62
                     └─> torch/_dynamo/symbolic_convert.py:54
                          └─> torch/_dynamo/exc.py:44 → utils.py:68
                               └─> torch.fx.experimental.symbolic_shapes.py:3
                                    └─> import sympy   ← CRASH
```

`@torch.compiler.disable` ist am **Modulstart** (`attention.py:39`) angehängt. Python führt Module-level-Decoratoren beim Import der Datei aus. `torch.compiler.disable` importiert sofort `torch._dynamo`, um die „disabled“-Registrierung zu setzen — und `torch._dynamo` lädt über `symbolic_shapes` zwingend `sympy` (am Import-Zeitpunkt, nicht lazy).

## Der Konflikt im Dockerfile
```dockerfile
# Verified: pocket_tts loads fine without sympy, networkx, pygments, pip, setuptools
RUN rm -rf /usr/local/lib/python3.13/site-packages/sympy \
           /usr/local/lib/python3.13/site-packages/sympy-*.dist-info \
```
`sympy` war als Transitiv-Dependency von `networkx` da (Zeile 40–41: `networkx` wird ebenfalls gelöscht) → korrekt als unnötig zu identifizieren, **wenn nichts `torch._dynamo` importiert**. Der Cleanup-Kommentar „pocket_tts loads fine without sympy“ gilt nur, solange kein `@torch.compiler.disable`-Decorator im Importpfad steckt. `pocket-tts` 2.1.0 hat diesen Decorator; die Annahme ist damit falsch.

## Warum das (vermutlich) vorher noch funktionierte
- Der Decorator wurde nach der letzten erfolgreichen Image-Verifikation hinzugefügt, **oder** `pocket-tts` wurde auf 2.1.0 hochgezogen (in `uv.lock`), ohne dass das Cleanup angepasst wurde.
- In älteren `torch`-Versionen wurde `torch._dynamo`/`symbolic_shapes` beim Import nicht am Import-Zeitpunkt geladen — in neueren schon.
- Der Crash passiert beim *Import*, also auch ohne Modellden — das „Waiting for Wyoming server to be ready“-Warten scheitert schon vor der eigentlichen Nutzung.

## Lösungsoptionen (nicht umgesetzt, zur Auswahl)
| # | Option | Aufwand | Image-Δ | Beschreibung |
|---|--------|---------|---------|--------------|
| A | **sympy wieder behalten** (empfohlen) | klein | +~3–4 MB | `sympy` + `sympy-*.dist-info` aus dem Cleanup-`rm` entfernen. Lokal, reversibel, kein Patch am upstream-Code. |
| B | Decorator patchen | mittel | +~3–4 MB | `pocket-tts/modules/attention.py` in der Docker-Build patchen, `@torch.compiler.disable` → `@torch.dynamo.disable` (oder ohne Decorator). `torch.dynamo.disable` ruft `torch._dynamo` *nicht* beim Import auf. |
| C | Torch-Import-Deaktivierung | mittel | +~3–4 MB | Environment-Variable oder Import-Hook, um `torch._dynamo` am Import-Zeitpunkt zu vermeiden. Versionsspezifisch, brüchig. |

> Hinweis: `networkx` ist *nicht* vom Crash betroffen — nur `sympy` (dessen Untermodul `torch.fx.experimental.symbolic_shapes` importiert). `networkx` kann also weiterhin gelöscht werden.

## Empfehlung
**Option A** — `sympy` wieder im Cleanup behalten. Minimal, sicher, reversibel, berührt keinen upstream-Code. `networkx` bleibt sauber entfernt.
