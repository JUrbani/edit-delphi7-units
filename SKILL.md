---
name: editar-archivos-cp1252
description: Editar archivos codificados en Windows-1252 / ANSI sin corromper acentos ni eñes. Usar SIEMPRE antes de tocar fuentes Delphi (.pas, .dfm, .dpr, .dpk, .inc) y ante cualquier archivo cuyo texto acentuado se vea como "Direcci?n", "art?culo" o con caracteres de reemplazo. También cubre aplicar .patch de otros equipos sin ensuciar el diff con finales de línea.
---

# Editar archivos Windows-1252 sin romper acentos

Aplica a fuentes en **Windows-1252 (CP1252), sin BOM y con CRLF** — típicamente
proyectos Delphi (`.pas`, `.dfm`, `.dpr`, `.dpk`, `.inc`). No son UTF-8.

**Cómo confirmarlo antes de editar:** si al leer el archivo ves `Direcci?n` o
`art?culo` donde debería haber acentos, o si `bytes>127` cae en el rango CP1252
(0xE1 0xE9 0xED 0xF3 0xFA 0xF1 0xBF 0xBA) en vez de venir en pares/tríos UTF-8,
es CP1252. Ver el bloque de verificación al final.

## La regla

**No uses las tools Edit ni Write sobre estos archivos.** Las decodifica como UTF-8:
cada byte acentuado (`á`=0xE1, `ó`=0xF3, `ñ`=0xF1, `¿`=0xBF, `º`=0xBA…) se vuelve
U+FFFD y al escribir queda `EF BF BD`. El daño es silencioso, alcanza a **todo** el
archivo —no solo a las líneas que tocaste— y el compilador no lo detecta: los
mensajes al usuario quedan con basura.

Editá siempre leyendo y escribiendo con encoding explícito por PowerShell.

## Patrón base: leer, modificar, escribir

```powershell
$enc  = [System.Text.Encoding]::GetEncoding(1252)
$path = 'C:\Fuentes\Neoretail\Client\...\Unidad.pas'
$s    = $enc.GetString([System.IO.File]::ReadAllBytes($path))

# ... transformar $s ...

[System.IO.File]::WriteAllBytes($path, $enc.GetBytes($s))
```

`Get-Content`/`Set-Content`/`Out-File` **no** sirven: reinterpretan el encoding y
normalizan finales de línea. Siempre `ReadAllBytes`/`WriteAllBytes`.

Para ver el archivo con los acentos correctos (Read también lo decodifica mal):

```powershell
$l = ($enc.GetString([System.IO.File]::ReadAllBytes($path)) -replace "`r","") -split "`n"
120..140 | ForEach-Object { "{0,4}: {1}" -f ($_+1), $l[$_] }
```

## Acentos dentro del script

Un `.ps1` que escribís con la tool Write queda en UTF-8, y PowerShell 5.1 sin BOM lo
lee como ANSI: los acentos que pongas literales en el script se corrompen antes de
llegar al archivo. Construilos por código:

```powershell
$A=[char]0xE1; $E=[char]0xE9; $I=[char]0xED; $O=[char]0xF3; $U=[char]0xFA
$N=[char]0xF1; $IQ=[char]0xBF; $NO=[char]0xBA

$texto = "El remito no est${A} en estado confirmado."   # here-string o interpolación
```

Cuidado al mapear: revisá que cada token apunte a la letra correcta antes de
escribir, y releé el resultado en pantalla. Un `${A}` donde iba `${I}` pasa
desapercibido en el diff.

## Reemplazo por texto exacto

Preferí `.Replace()` sobre bloques literales, con guarda previa:

```powershell
function NL([string]$t) { $t -replace "`r?`n", "`r`n" }   # here-strings a CRLF

$old = NL @"
  if lStatus <> 'C' then
    Exit;
"@
if (-not $s.Contains($old)) { throw "no match: el ancla cambió" }
$s = $s.Replace($old, $new)
```

El `throw` es lo que evita el peor caso: que el script "ande" y no toque nada.

## Reemplazo por rangos de línea

Para bloques grandes, splice sobre una lista, **de abajo hacia arriba** para que no
se corran los índices:

```powershell
$lines = New-Object System.Collections.Generic.List[string]
(($s -replace "`r","") -split "`n") | ForEach-Object { $lines.Add($_) }

# verificar anclas ANTES de tocar nada (1-based -> índice = L-1)
$anchors = @{ 667 = '//m.b TT30198'; 870 = '//fin' }
foreach ($k in $anchors.Keys) {
  if ($lines[$k-1] -ne $anchors[$k]) { throw "ancla $k : [$($lines[$k-1])]" }
}

$lines.RemoveRange(664, 871-665+1)
$lines.InsertRange(664, [string[]]$nuevo)

[System.IO.File]::WriteAllBytes($path, $enc.GetBytes(($lines -join "`r`n")))
```

Para **extraer** código existente (refactors), capturá las líneas del array y
reinsertalas tal cual en vez de retipearlas: garantiza extracción byte a byte y
mantiene el diff limpio.

```powershell
$body = $lines[1922..2014]     # capturar ANTES de cualquier RemoveRange
```

## Trampa de PowerShell: concatenar arrays

`Detoken @(...) + $otro + @(...)` se parsea como argumentos del comando, no como
concatenación: perdés silenciosamente todo menos el primer trozo. Usá variables
intermedias y armá el resultado explícitamente:

```powershell
$out = New-Object System.Collections.Generic.List[string]
foreach ($chunk in @($parte1, $parte2, $parte3)) {
  foreach ($ln in $chunk) { $out.Add([string]$ln) }
}
```

## Verificación obligatoria al terminar

```powershell
$b = [System.IO.File]::ReadAllBytes($path); $crlf=0; $lf=0; $bad=0; $hi=0
for ($i=0; $i -lt $b.Length; $i++) {
  if ($b[$i] -eq 10) { if ($i -gt 0 -and $b[$i-1] -eq 13) { $crlf++ } else { $lf++ } }
  if ($b[$i] -gt 127) { $hi++ }
  if ($b[$i] -eq 0xEF -and $i+2 -lt $b.Length -and $b[$i+1] -eq 0xBF -and $b[$i+2] -eq 0xBD) { $bad++ }
}
"CRLF=$crlf LFsueltos=$lf bytes>127=$hi corruptos=$bad"
```

Se espera: `LFsueltos=0`, `corruptos=0`, y `bytes>127` en el rango de CP1252
(0xBA 0xBF 0xC1 0xC9 0xD1 0xE1 0xE9 0xED 0xF1 0xF3 0xFA…). Si aparecen `239,191,189`
(0xEF 0xBF 0xBD) el archivo ya está roto: revertí con `git checkout -- <archivo>` y
rehacé la edición.

Comparar contra el original también sirve de red:

```powershell
cmd /c "git cat-file blob HEAD:ruta/con/barras/Unidad.pas > %TEMP%\orig.pas"
```

`git show ... > archivo` desde PowerShell **reencodea**; por eso va vía `cmd /c`.

## Aplicar un .patch de otro equipo

Si el otro entorno tiene `core.autocrlf=false`, sus líneas agregadas viajan con CRLF
mientras el índice guarda LF, y git marca el archivo entero como cambiado. Normalizá
el patch a LF antes de aplicarlo:

```powershell
$b = [System.IO.File]::ReadAllBytes($src); $out = New-Object System.Collections.Generic.List[byte]
for ($i=0; $i -lt $b.Length; $i++) {
  if ($b[$i] -eq 13 -and ($i+1) -lt $b.Length -and $b[$i+1] -eq 10) { continue }
  $out.Add($b[$i])
}
[System.IO.File]::WriteAllBytes($dst, $out.ToArray())
git apply --whitespace=nowarn --include=<ruta> $dst
```

`git apply` es byte-exacto y **no** rompe el encoding: si aparecen caracteres
corruptos después de aplicar un patch, los introdujo una edición posterior, no git.

Ojo con `--whitespace=fix`: si el repo ya tenía espacios al final de línea, los saca
en todo el archivo y te infla el diff con ruido. Usá `--whitespace=nowarn`.

## Chequeo estructural post-edición (Delphi)

No hay compilador a mano, así que compará el balance `begin`/`end` contra el original;
la diferencia debe mantenerse salvo por lo que agregaste a propósito (un `try/finally`
o una declaración de clase suman `end` sin `begin`):

```powershell
function Bal($text) {
  $b=0; $e=0
  foreach ($ln in ($text -replace "`r","") -split "`n") {
    $t = $ln -replace "'[^']*'","" -replace "//.*$",""    # sin literales ni comentarios
    $b += ([regex]::Matches($t,'(?i)(?<![A-Za-z0-9_])begin(?![A-Za-z0-9_])')).Count
    $e += ([regex]::Matches($t,'(?i)(?<![A-Za-z0-9_])end(?![A-Za-z0-9_])')).Count
  }
  return @($b,$e)
}
```
