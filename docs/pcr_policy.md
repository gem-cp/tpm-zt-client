# PCR Policy

Um einen TPM-Schlüssel zu erstellen, der für alle Benutzer des Systems verwendbar ist, aber auf eine bestimmte Anwendung eingeschränkt wird, müssen Sie eine Kombination von TPM-Attributen und Mechanismen wie **Authorisierung**, **Schlüsselschutz**, und **Anwendungsintegration** verwenden. Hier sind die Schritte, um dies zu erreichen:

### 1. **Schlüssel mit einer Policy binden**
Eine Möglichkeit, den Zugriff auf den Schlüssel zu beschränken, ist die Verwendung einer **Policy**, die festlegt, unter welchen Bedingungen der Schlüssel verwendet werden kann. Sie können eine Policy erstellen, die beispielsweise einen spezifischen PCR-Wert erfordert, der durch die Anwendung gesetzt wird.

- **PCR-Policy**: Die Policy könnte an einen bestimmten Zustand des Systems gebunden sein, der durch PCRs (Platform Configuration Registers) im TPM dargestellt wird. Die Anwendung könnte beim Start einen bestimmten Wert in eine PCR schreiben, und der Schlüssel ist nur nutzbar, wenn diese PCR den erwarteten Wert hat.

### 2. **Schlüssel-Attributkonfiguration**
Beim Erstellen des Schlüssels können bestimmte Attribute gesetzt werden, die definieren, wie und wann der Schlüssel verwendet werden kann:

- **`fixedTPM` und `fixedParent`**: Diese Attribute verhindern, dass der Schlüssel außerhalb des TPMs exportiert oder unter einem anderen Elternschlüssel verwendet wird.
  
- **`userWithAuth`**: Dieses Attribut erzwingt, dass der Schlüssel mit einer Authorisierung verwendet wird, die bei der Erstellung festgelegt wurde. Dies könnte eine Policy sein, die an die Anwendung gebunden ist.

- **`restricted`**: Ein eingeschränkter Schlüsseltyp könnte verwendet werden, um sicherzustellen, dass der Schlüssel nur für bestimmte TPM-Operationen wie Signatur- oder Entschlüsselungsvorgänge verwendet wird.

### 3. **Anwendungsspezifische Integration**
Die Anwendung selbst kann so gestaltet werden, dass sie den Schlüssel verwendet, indem sie die nötige Policy setzt und die korrekten TPM-Operationen durchführt.

### Beispiel: Schlüssel mit PCR-Policy binden

Hier ein Beispiel, wie man einen TPM-Schlüssel erstellt, der an eine PCR-Policy gebunden ist. Dies bedeutet, dass der Schlüssel nur verwendet werden kann, wenn die PCRs in einem bestimmten Zustand sind, den nur die Anwendung setzen kann.

```python
from tpm2_pytss import *
from tpm2_pytss.binding import *

# TPM Kontext initialisieren
with TctiLdr('mssim') as tcti:
    with EsysContext(tcti) as esys_ctx:
        
        # PCR Auswahl (z.B. PCR 0)
        pcr_selection = TPML_PCR_SELECTION(
            count=1,
            pcrSelections=[
                TPMS_PCR_SELECTION(
                    hash=TPM2_ALG.SHA256,
                    sizeofSelect=3,
                    pcrSelect=[1, 0, 0]
                )
            ]
        )
        
        # PCR Werte holen, um die Policy zu erstellen
        pcr_digest = esys_ctx.PCR_Read(pcr_selection)[0]

        # Policy Digest auf Basis der PCR
        policy_digest = esys_ctx.PolicyPCR(policySession=None, pcrDigest=pcr_digest, pcrs=pcr_selection)

        # Sensitiv-Daten zur Schlüsselerstellung
        sensitive = TPM2B_SENSITIVE_CREATE(
            sensitive=TPMS_SENSITIVE_CREATE(
                userAuth=TPM2B_AUTH(size=0, buffer=[]),
                data=TPM2B_SENSITIVE_DATA(size=0, buffer=[])
            )
        )

        # Public Key Parameters inklusive Policy
        public = TPM2B_PUBLIC(
            publicArea=TPMT_PUBLIC(
                type=TPM2_ALG.RSA,
                nameAlg=TPM2_ALG.SHA256,
                objectAttributes=(TPMA_OBJECT.SIGN_ENCRYPT |
                                  TPMA_OBJECT.FIXEDTPM |
                                  TPMA_OBJECT.FIXEDPARENT |
                                  TPMA_OBJECT.SENSITIVEDATAORIGIN |
                                  TPMA_OBJECT.USERWITHAUTH),
                authPolicy=policy_digest,
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

        # Schlüssel mit der PCR-Policy erstellen
        primary_handle = esys_ctx.CreatePrimary(
            primaryHandle=TPM2_RH.OWNER,
            inSensitive=sensitive,
            inPublic=public,
            outsideInfo=TPM2B_DATA(size=0, buffer=[]),
            creationPCR=TPML_PCR_SELECTION(count=0),
        )

        print("Schlüssel mit PCR Policy erstellt, Handle:", primary_handle)
```

### Schritte zur Umsetzung:
1. **PCR-Policy erstellen**: Die Policy sollte so gestaltet sein, dass sie an eine bestimmte PCR gebunden ist, die durch die Anwendung gesteuert wird.
2. **Schlüssel erstellen**: Der Schlüssel wird mit dieser Policy erstellt, so dass er nur dann verwendet werden kann, wenn die Policy erfüllt ist.
3. **Anwendung konfigurationsfähig machen**: Die Anwendung sollte die Fähigkeit haben, den Zustand der PCRs zu beeinflussen oder sicherzustellen, dass die Policy erfüllt ist, bevor der Schlüssel verwendet wird.

### Zusammengefasst:
- **Anwendungsbindung**: Durch die Verwendung von PCRs oder spezifischen Policies kann sichergestellt werden, dass der Schlüssel nur von einer bestimmten Anwendung verwendet werden kann, obwohl er für alle Benutzer zugänglich ist.
- **Schlüsselzugriff**: Der Schlüssel wird durch TPM-Policies geschützt, die sicherstellen, dass nur autorisierte Anwendungen ihn verwenden können, unabhängig davon, welcher Benutzer die Anwendung ausführt.