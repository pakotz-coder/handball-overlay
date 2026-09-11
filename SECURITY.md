# Plan de securitate — Handball Overlay Suite

Audit al modelului de securitate al aplicației (`index.html`, `control.html`, `output.html`
+ Firebase Realtime Database) și pașii de întărire. Analog cu auditul de izolare de la
aplicația de handbal: acolo izolarea se face pe server (`WHERE liga=?`), aici trebuie făcută
prin **regulile Firebase + secretul room-ului**.

## 1. Modelul de securitate (cum funcționează azi)

Întreaga încredere se bazează pe un singur lucru: **`rooms/{room}` în Firebase RTDB**, unde
`room` este derivat dintr-un nume ales de utilizator, cu **autentificare anonimă**.

- `index.html` → generează room-ul din nume (`[^a-z0-9]` eliminat).
- `control.html` → scrie în `rooms/{room}` (comenzi, match, culori, opacitate, limbă).
- `output.html` → **citește** din `rooms/{room}` (overlay-ul din OBS/Yolobox). Nu se
  autentifică — depinde de `.read: true`.

Cine cunoaște URL-ul (deci room-ul) poate controla transmisiunea. Room-ul = parola.

## 2. Constatări

### 🔴 CRITIC — room-uri ghicibile → deturnarea transmisiunii
Regula de scriere este `".write": "auth != null"`, iar aplicația **loghează anonim orice
vizitator**. Practic `auth != null` este adevărat pentru **oricine**. Singura barieră este
secretul room-ului — dar numele erau scurte și predictibile (ex. `Mario7563`). Oricine
ghicește sau vede un room poate afișa overlay-uri false sau da `Clear` peste o transmisiune
live.

**Remediu (aplicat):** `index.html` generează acum room-uri cu **token aleator lung** (nume +
~12 caractere din `crypto.getRandomValues`, ~60+ biți de entropie), persistate per-nume în
`localStorage`. Room-urile nu pot fi enumerate (`/rooms` are `.read: false`), deci un atacator
ar trebui să ghicească token-ul întreg — infezabil.

### 🟠 MEDIU — scrieri fără validare (abuz / cost)
`auth != null` permitea scrierea **oricărei** structuri, de orice dimensiune, sub orice cheie.
Un utilizator autentificat putea umple baza cu date arbitrare.

**Remediu (în `database.rules.json`):** validări pe fiecare nod (`opacity` 0–100, `lang`
en/ro, `command.type` show/clear, limite de lungime), plus `"$other": { ".validate": false }`
care respinge chei necunoscute la nivel de room și de comandă.

### 🟠 MEDIU — `new Function()` pe date de la terț
`control.html:167` evaluează răspunsul brut de la `sportinfocentar2.com` prin
`new Function('return('+txt+')')`. Există o gardă (întâi `JSON.parse`, apoi un regex de cuvinte
interzise), dar rămâne cod remote executat.

**Recomandare:** dacă răspunsul e mereu JSON valid, elimină complet ramura `new Function` și
lasă doar `JSON.parse`. Dacă terțul chiar trimite JS non-JSON, întărește regex-ul și
loghează cazurile. (Neaplicat încă — necesită confirmarea formatului real al răspunsului.)

### 🟢 OK — XSS pe datele scrapate
Datele de meci sunt trecute prin `sanitize()` (escape HTML) la scriere și la citire
(`output.html:729`). Injecția de HTML prin numele jucătorilor este acoperită.

### 🟢 OK / de reținut — cheia Firebase publică
`apiKey` din config este **publică prin design** la aplicațiile Firebase de tip client. Nu e o
scurgere. Protecția reală o dau **regulile RTDB**, nu cheia.

## 3. Ce rămâne de făcut (nu se poate automatiza din repo)

1. **Publică regulile noi în Firebase.** Deschide Firebase Console → Realtime Database → Rules,
   lipește conținutul din [`database.rules.json`](database.rules.json), apasă **Publish**.
   Nu se poate face din cod — necesită accesul tău la consolă.
2. (Opțional, întărire mai puternică) Restricționează `.read` la `auth != null` — dar atunci
   trebuie adăugat `firebase.auth().signInAnonymously()` și în `output.html`, altfel overlay-ul
   din OBS nu mai citește. Câștig mic (room-urile oricum nu se pot enumera), deci opțional.
3. (Opțional) Separă capabilitatea de **citire** (output) de cea de **scriere** (control) prin
   două id-uri diferite, ca output-ul pus în OBS să nu confere și drept de scriere. Refactor mai
   mare; de evaluat dacă expunerea URL-ului de output devine o problemă.

## 4. Rezumatul modificărilor din acest audit

| Fișier | Modificare |
|---|---|
| `database.rules.json` | **Nou.** Reguli întărite: validări per-nod, respingere chei necunoscute, format room. |
| `index.html` | Room generat cu token aleator secret + persistență per-nume; text instrucțiune actualizat. |
| `SECURITY.md` | **Nou.** Acest document. |

`control.html` și `output.html` nu necesită modificări: sanitizarea și citirea publică rămân
corecte. Nimic nu este „live" până nu publici regulile din pasul 3.1.
