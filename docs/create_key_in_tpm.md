# Schlüssel im TPM erzeugen

Um einen eigenen Signaturschlüssel in das Trusted Platform Module (TPM) eines Windows-Computers einzutragen, können wir die `tpm2-pytss`-Bibliothek verwenden, die eine Python-Schnittstelle zur TCG TPM 2.0 TSS (Trusted Software Stack) API bietet. Damit können wir auf TPM-Funktionen zugreifen und Schlüssel generieren, speichern und verwenden.

Hier ist ein Python-Beispielcode, der zeigt, wie ein RSA-Schlüsselpaar im TPM erzeugt und gespeichert wird:

```python
from tpm2_pytss import *
from tpm2_pytss.binding import *

# TCTI (Transmission Interface) initialisieren
with TctiLdr('mssim') as tcti:
    # TPM-Kontext erstellen
    with EsysContext(tcti) as esys_ctx:
        
        # Key Parameters
        public = TPM2B_PUBLIC(
            publicArea=TPMT_PUBLIC(
                type=TPM2_ALG.RSA,
                nameAlg=TPM2_ALG.SHA256,
                objectAttributes=(TPMA_OBJECT.SIGN_ENCRYPT |
                                  TPMA_OBJECT.FIXEDTPM |
                                  TPMA_OBJECT.FIXEDPARENT |
                                  TPMA_OBJECT.SENSITIVEDATAORIGIN |
                                  TPMA_OBJECT.USERWITHAUTH),
                parameters=TPMS_RSA_PARMS(
                    symmetric=TPMT_SYM_DEF_OBJECT(algorithm=TPM2_ALG.NULL),
                    scheme=TPMT_RSA_SCHEME(scheme=TPM2_ALG.RSASSA,
                                           details=TPMU_SIG_SCHEME(rsassa=TPMS_SIG_SCHEME_RSASSA(hashAlg=TPM2_ALG.SHA256))),
                    keyBits=2048,
                    exponent=0,
                ),
                unique=TPMU_PUBLIC_ID(rsa=TPM2B_PUBLIC_KEY_RSA(size=0, buffer=[]))
            )
        )

        sensitive = TPM2B_SENSITIVE_CREATE(
            sensitive=TPMS_SENSITIVE_CREATE(
                userAuth=TPM2B_AUTH(size=0, buffer=[]),
                data=TPM2B_SENSITIVE_DATA(size=0, buffer=[])
            )
        )

        # Erstellen eines Primary Key (z.B. in der Endorsement-Hierarchie)
        primary_handle = esys_ctx.CreatePrimary(
            primaryHandle=TPM2_RH.ENDORSEMENT,
            inSensitive=sensitive,
            inPublic=public,
            outsideInfo=TPM2B_DATA(size=0, buffer=[]),
            creationPCR=TPML_PCR_SELECTION(count=0),
        )

        # Der erzeugte Schlüssel kann nun verwendet oder gespeichert werden.
        print("Schlüssel erfolgreich im TPM erstellt. Handle:", primary_handle)

        # Hier könnte man den Schlüssel persistent im TPM speichern, falls nötig:
        # handle = esys_ctx.EvictControl(auth=TPM2_RH.OWNER, objectHandle=primary_handle, persistentHandle=0x81010001)
```

### Voraussetzungen:

1. **Python-Umgebung**: Sie benötigen eine Python-Umgebung auf einem Windows-Rechner.

2. **Installation von Abhängigkeiten**:
   - Installieren Sie `tpm2-pytss` (der Python-Bindings für TSS):
   ```bash
   pip install tpm2-pytss
   ```

3. **TPM 2.0**: Der Computer muss TPM 2.0 aktiviert haben, und der TPM-Software-Stack muss installiert sein. Das TPM muss auch entsprechend initialisiert sein (z. B. durch Windows-Bootprozess oder manuell).

### Anmerkungen:

- Der Code erstellt einen RSA-Primärschlüssel in der Endorsement-Hierarchie und gibt den TPM-Handle des Schlüssels aus. Dieser Handle kann für Signaturoperationen oder andere TPM-Operationen verwendet werden.
  
- Wenn Sie den Schlüssel persistent speichern möchten, können Sie die `EvictControl`-Methode verwenden, um ihn an einem bestimmten persistenten Handle (z. B. `0x81010001`) zu speichern.

Beachten Sie, dass dieser Code eher für Entwicklung und Testzwecke geeignet ist und vor dem Einsatz in einer Produktionsumgebung eine gründliche Überprüfung und Sicherheitsüberlegungen erforderlich sind.

In der Regel benötigen Sie Administratorrechte, um auf das TPM eines Windows-Computers zuzugreifen und Operationen wie das Erstellen und Speichern von Schlüsseln durchzuführen. Der Zugriff auf das TPM erfordert erhöhte Berechtigungen aus folgenden Gründen:

1. **TPM-Sicherheit**: Das TPM ist ein sicherheitskritischer Bestandteil des Systems, und der Zugriff darauf ist stark reglementiert, um sicherzustellen, dass nur autorisierte Benutzer sicherheitsrelevante Operationen durchführen können.

2. **TPM-Treiber und -Schnittstellen**: Um mit dem TPM zu kommunizieren, muss der Python-Code über Treiber und Schnittstellen laufen, die typischerweise Administratorrechte erfordern. 

3. **Persistente Speicherung von Schlüsseln**: Wenn Sie einen Schlüssel persistent im TPM speichern möchten (z.B. durch `EvictControl`), erfordert dies ebenfalls Administratorrechte, um Änderungen an der TPM-Konfiguration vorzunehmen.