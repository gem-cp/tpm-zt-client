Um die Version des Betriebssystems, die Installation und Aktivierung eines Virenscanners sowie dessen Version und den Status der Firewall auf einem Windows-System zu ermitteln, können Sie die `platform`, `subprocess`, und `winreg` Module verwenden. Hier ist ein Python-Skript, das dies ausliest:

```python
import platform
import subprocess
import winreg as reg

# Funktion, um die Windows-Version zu ermitteln
def get_windows_version():
    version = platform.version()
    release = platform.release()
    os_version = platform.win32_ver()
    return f"Windows Version: {os_version[0]}, Build: {version}, Release: {release}"

# Funktion, um den Status des Virenscanners zu ermitteln
def get_antivirus_status():
    try:
        cmd = 'wmic /namespace:\\\\root\\SecurityCenter2 path AntiVirusProduct get displayName,productState,versionNumber /format:list'
        output = subprocess.check_output(cmd, shell=True).decode()
        av_info = [line.strip() for line in output.splitlines() if line.strip()]
        if not av_info:
            return "No Antivirus found or antivirus information could not be retrieved."
        
        av_status = {}
        for info in av_info:
            key, value = info.split('=')
            av_status[key.strip()] = value.strip()

        state = av_status.get("productState")
        if state:
            state = int(state, 16)
            enabled = state & 0x10 > 0
            running = state & 0x100000 > 0
            state_desc = "enabled and running" if enabled and running else "disabled or not running"
            return f"Antivirus: {av_status.get('displayName')} (Version: {av_status.get('versionNumber')}) is {state_desc}."
        else:
            return "Unable to determine antivirus state."
    except subprocess.CalledProcessError as e:
        return f"Error retrieving antivirus status: {str(e)}"

# Funktion, um den Status der Firewall zu ermitteln
def get_firewall_status():
    try:
        cmd = 'netsh advfirewall show allprofiles'
        output = subprocess.check_output(cmd, shell=True).decode()
        if "State ON" in output:
            return "Firewall is enabled."
        elif "State OFF" in output:
            return "Firewall is disabled."
        else:
            return "Unable to determine firewall status."
    except subprocess.CalledProcessError as e:
        return f"Error retrieving firewall status: {str(e)}"

if __name__ == "__main__":
    print(get_windows_version())
    print(get_antivirus_status())
    print(get_firewall_status())
```

### Erklärung des Codes:
1. **Windows-Version**:
   - `platform.version()`, `platform.release()`, und `platform.win32_ver()` werden verwendet, um die Windows-Version und den Build zu ermitteln.

2. **Antivirus-Status**:
   - `wmic` (Windows Management Instrumentation Command-line) wird verwendet, um Informationen über den installierten Virenscanner abzurufen.
   - Der `productState`-Wert gibt den Status des Virenscanners an (ob er aktiv und läuft).
   - `productState` wird als Hexadezimalwert interpretiert, wobei bestimmte Bits anzeigen, ob der Scanner aktiviert und in Betrieb ist.

3. **Firewall-Status**:
   - Der `netsh advfirewall show allprofiles`-Befehl zeigt den Status der Windows-Firewall für alle Profile (Domain, Private, Public) an.
   - Der Code durchsucht den Output nach "State ON" oder "State OFF", um den Firewall-Status zu bestimmen.

### Ausführung:
- Dieses Skript kann von einem normalen Benutzer ausgeführt werden, erfordert aber möglicherweise erhöhte Rechte, um genaue Informationen über den Virenscanner und die Firewall zu erhalten.
  
- In einem Produktionsumfeld sollten Sie sicherstellen, dass das Skript auf Systemen mit ausreichenden Rechten ausgeführt wird, um die Informationen korrekt abzurufen.