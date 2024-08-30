# TPM Attestation

Der Nachweis, dass ein Signaturschlüssel aus einem TPM stammt, kann auf mehrere Weisen erfolgen. Der Nachweis basiert auf dem Vertrauensmodell, das TPMs zugrunde liegt, und nutzt spezifische Eigenschaften des TPM, um die Herkunft der Signatur und des Schlüssels zu bestätigen. Hier sind einige Methoden, um dies nachzuweisen:

### 1. **Attestation (Quote)**
Attestation ist ein Mechanismus, bei dem das TPM selbst eine signierte Aussage (Quote) über den Zustand und die Eigenschaften des Systems (einschließlich der verwendeten Schlüssel) macht. Ein Verifizierer kann diese Attestation überprüfen, um sicherzustellen, dass der Schlüssel im TPM erstellt wurde.

#### Beispielvorgehen:
1. **PCRs (Platform Configuration Registers)**: TPMs halten Werte in PCRs, die den Systemzustand widerspiegeln. Eine TPM-Quote kann Informationen über diese PCRs und den Schlüssel beinhalten.
   
2. **TPM Quote erstellen**:
   - Sie erstellen eine Quote mit dem TPM, die durch einen speziellen Attestationsschlüssel (oft als AIK, Attestation Identity Key) signiert ist.
   - Diese Quote enthält Hashes von PCR-Werten, die durch den TPM-Wert und den Schlüssel selbst beeinflusst werden können.

3. **Verifizierung**:
   - Der Verifizierer (z.B. ein Server oder ein anderer TPM) überprüft die Signatur der Quote und stellt sicher, dass die Signatur nur von einem TPM erzeugt werden konnte.
   - Der Verifizierer kann auch den Zustand des Systems anhand der PCR-Werte beurteilen.

### 2. **Schlüsselzertifikate**
Einige TPMs erlauben die Generierung eines Schlüssels mit einem Zertifikat, das durch eine vertrauenswürdige Zertifizierungsstelle (CA) ausgestellt wurde. Dieses Zertifikat kann die Herkunft des Schlüssels bestätigen und belegen, dass der Schlüssel im TPM generiert wurde.

#### Beispielvorgehen:
1. **Schlüsselgenerierung**: Ein TPM generiert einen Schlüssel und kann, basierend auf einem AIK, ein Zertifikat für diesen Schlüssel anfordern.
   
2. **Zertifikatsanfrage**: Eine vertrauenswürdige Zertifizierungsstelle, die das TPM vertraut, stellt ein Zertifikat aus, das bestätigt, dass der Schlüssel innerhalb eines bestimmten TPM erstellt wurde.

3. **Verifizierung**: Jeder, der eine Signatur verifizieren will, kann das Zertifikat des Schlüssels prüfen, um sicherzustellen, dass der Schlüssel aus einem TPM stammt.

### 3. **TPM-spezifische Attribute**
Ein TPM kann bestimmte Attribute oder Metadaten zu einem Schlüssel beifügen, die spezifisch für das TPM und seine Umgebung sind. Diese Attribute können genutzt werden, um die Herkunft des Schlüssels nachzuweisen.

#### Beispielvorgehen:
- **Lesen der Attribute**: Prüfen Sie die Attribute des Schlüssels, die während der Schlüsselerstellung im TPM gesetzt wurden. Diese Attribute könnten spezifische Flags oder Eigenschaften enthalten, die darauf hinweisen, dass der Schlüssel nur im TPM verwendbar ist.
- **Verifizierung**: Diese Informationen können überprüft werden, um sicherzustellen, dass der Schlüssel aus einem TPM stammt.

### Beispiel für TPM-Quote in Python

Hier ein Codebeispiel, wie eine Quote mit `tpm2-pytss` erstellt werden könnte:

```python
from tpm2_pytss import *
from tpm2_pytss.binding import *

# TCTI (Transmission Interface) initialisieren
with TctiLdr('mssim') as tcti:
    # TPM-Kontext erstellen
    with EsysContext(tcti) as esys_ctx:
        
        # PCR Selektion (z.B. PCR 0 und 1)
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

        # Quoten eines PCR-Wertes
        quoted, signature = esys_ctx.Quote(
            signHandle=0x81010001,  # Handle des Schlüssels
            qualifyingData=TPM2B_DATA(size=0, buffer=[]),
            inScheme=TPMT_SIG_SCHEME(
                scheme=TPM2_ALG.RSASSA,
                details=TPMU_SIG_SCHEME(rsassa=TPMS_SIG_SCHEME_RSASSA(hashAlg=TPM2_ALG.SHA256))
            ),
            PCRselect=pcr_selection
        )

        print("Quote erstellt:", quoted)
        print("Signatur der Quote:", signature)
```

### Fazit:
Um den Nachweis zu erbringen, dass ein Signaturschlüssel aus einem TPM stammt, sind Attestation (Quote), Schlüsselzertifikate und TPM-spezifische Attribute die wichtigsten Methoden. Diese Methoden können kombiniert verwendet werden, um eine robuste Bestätigung zu liefern, dass die Signatur und der Schlüssel tatsächlich aus einem vertrauenswürdigen TPM stammen.