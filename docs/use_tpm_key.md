# Einsatz eines TPM Schlüssels

Nachdem der Schlüssel im TPM erstellt und (gegebenenfalls) persistent gespeichert wurde, können Sie als normaler Benutzer den Schlüssel verwenden, um eine Signatur zu erzeugen, sofern Sie Zugriff auf den Schlüssel haben und die erforderlichen Berechtigungen für die TPM-Operationen bestehen. Hier ist ein Beispielcode, der zeigt, wie Sie mit einem bestehenden Schlüssel im TPM eine Signatur erzeugen können:

```python
from tpm2_pytss import *
from tpm2_pytss.binding import *
import hashlib

# TCTI (Transmission Interface) initialisieren
with TctiLdr('mssim') as tcti:
    # TPM-Kontext erstellen
    with EsysContext(tcti) as esys_ctx:
        
        # Handle des Schlüssels im TPM (z.B. ein persistenter Handle)
        persistent_handle = 0x81010001  # Beispielhandle; muss mit dem persistenten Handle des Schlüssels übereinstimmen

        # Daten, die signiert werden sollen
        data_to_sign = b"Dies sind die Daten, die signiert werden sollen."
        
        # Hash der Daten erzeugen (z.B. SHA-256)
        hash_object = hashlib.sha256()
        hash_object.update(data_to_sign)
        digest = hash_object.digest()

        # TPM2B_DIGEST Struktur erstellen
        digest_value = TPM2B_DIGEST(size=len(digest), buffer=list(digest))
        
        # Signatur erstellen
        signature = esys_ctx.Sign(
            keyHandle=persistent_handle,
            digest=digest_value,
            inScheme=TPMT_SIG_SCHEME(
                scheme=TPM2_ALG.RSASSA,
                details=TPMU_SIG_SCHEME(rsassa=TPMS_SIG_SCHEME_RSASSA(hashAlg=TPM2_ALG.SHA256))
            ),
            validation=None
        )

        print("Signatur erfolgreich erstellt:", signature)

        # Die Signatur kann nun weiterverwendet oder gespeichert werden.
```

### Erklärung des Codes:

- **Persistent Handle**: Der `persistent_handle` muss mit dem Handle des Schlüssels übereinstimmen, der zuvor im TPM gespeichert wurde. Wenn der Schlüssel nicht persistent gespeichert wurde, können Sie den temporären Handle verwenden, den Sie beim Erstellen des Schlüssels erhalten haben.

- **Signaturprozess**: Der Code hashiert die zu signierenden Daten (in diesem Fall mit SHA-256) und sendet den Hash dann an das TPM, um eine Signatur mit dem gespeicherten Schlüssel zu erzeugen.

- **TCTI (Transmission Interface)**: Das Beispiel verwendet `mssim`, das ein Software-Simulator ist. Auf einem echten TPM-System verwenden Sie normalerweise `device` oder eine andere TCTI, die für Ihre Umgebung geeignet ist.

### Ausführung als normaler Benutzer:

- **Signieren**: Der Signaturprozess erfordert in der Regel keine Administratorrechte, wenn der Schlüssel bereits im TPM erstellt und (falls erforderlich) persistent gespeichert wurde. Ein normaler Benutzer kann diesen Code ausführen, vorausgesetzt, der Benutzer hat Zugriff auf den Schlüssel im TPM.

- **Einschränkungen**: Der Benutzer benötigt Berechtigungen, um auf den spezifischen TPM-Schlüssel zuzugreifen. Falls der Schlüssel speziell für einen Benutzer oder eine Anwendung eingeschränkt ist, könnte der Zugriff verweigert werden.

### Zusammengefasst:
- **Erstellung des Schlüssels**: Erfordert Administratorrechte.
- **Verwendung des Schlüssels zur Signatur**: Kann in den meisten Fällen als normaler Benutzer durchgeführt werden, wenn der Schlüssel im TPM gespeichert ist und der Benutzer Berechtigungen hat.