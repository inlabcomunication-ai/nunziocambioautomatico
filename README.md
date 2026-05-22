# PN Autofficina · Landing + Dashboard

Setup completo per landing page con form contatti e dashboard amministrativa, tutto appoggiato su Firebase (Firestore + Auth + Hosting).

## Contenuto

- `index.html` — landing pubblica con form
- `dashboard.html` — area riservata per vedere/gestire i contatti
- `logo.png` — logo PN
- `firestore.rules` — regole di sicurezza per Firestore

---

## Setup Firebase (una volta sola, ~15 minuti)

### 1. Crea il progetto Firebase

1. Vai su https://console.firebase.google.com
2. Clicca **Aggiungi progetto** → nome a piacere (es. `pn-autofficina`)
3. Disabilita Google Analytics (non serve) e crea il progetto

### 2. Abilita Firestore

1. Nel menu laterale: **Build → Firestore Database**
2. Clicca **Crea database**
3. Modalità: **produzione** (le regole le impostiamo dopo)
4. Località: `europe-west` (es. `europe-west3` Francoforte)

### 3. Abilita Authentication

1. Menu laterale: **Build → Authentication**
2. Clicca **Inizia**
3. Tab **Metodo di accesso** → abilita **Email/password**
4. Tab **Utenti** → **Aggiungi utente** → inserisci email e password
   - **Questa sarà la credenziale per entrare nella dashboard.** Usane una forte.
   - Esempio: `nunzio@pnautofficina.it` / password lunga
   - Non c'è registrazione pubblica: solo gli utenti creati qui possono accedere.

### 4. Recupera la config

1. Menu laterale: **⚙ Impostazioni progetto** → **Generali**
2. Scorri fino a **Le tue app** → clicca l'icona web `</>`
3. Soprannome app: `landing` (o quello che vuoi), **non** abilitare Firebase Hosting per ora
4. Registra app
5. Copia il blocco `firebaseConfig` che ti mostra, esempio:

```js
const firebaseConfig = {
  apiKey: "AIzaSyXXXXXXXXXXXXXXXXXXXXXXXX",
  authDomain: "pn-autofficina.firebaseapp.com",
  projectId: "pn-autofficina",
  storageBucket: "pn-autofficina.appspot.com",
  messagingSenderId: "123456789012",
  appId: "1:123456789012:web:abc123def456"
};
```

### 5. Incolla la config nei file

Apri **`index.html`** e cerca `INSERISCI_API_KEY` (è in fondo al file, nello script `type="module"`). Sostituisci tutti e 6 i valori `INSERISCI_*` con quelli reali.

Stessa identica cosa in **`dashboard.html`**: la config va incollata anche lì, identica.

### 6. Imposta le regole Firestore

1. Console Firebase → **Firestore Database** → tab **Regole**
2. Cancella tutto e incolla il contenuto di `firestore.rules` (vedi sotto)
3. Clicca **Pubblica**

Regole (`firestore.rules`):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /leads/{leadId} {
      // Chiunque può creare un lead dal form pubblico
      allow create: if
        request.resource.data.keys().hasOnly(
          ['name','phone','email','plate','notes','status','createdAt','source']
        ) &&
        request.resource.data.name is string &&
        request.resource.data.name.size() > 0 &&
        request.resource.data.name.size() < 200 &&
        request.resource.data.phone is string &&
        request.resource.data.phone.size() > 0 &&
        request.resource.data.phone.size() < 50 &&
        request.resource.data.status == 'nuovo';

      // Solo utenti autenticati possono leggere/modificare/eliminare
      allow read, update, delete: if request.auth != null;
    }
  }
}
```

Cosa fanno queste regole:
- chiunque (form pubblico) può **creare** un lead, ma solo con i campi previsti e con `status: 'nuovo'` — impedisce a script malevoli di scrivere campi arbitrari o stati anomali
- solo chi è loggato può **leggere/aggiornare/cancellare**, quindi i contatti non sono pubblicamente accessibili

---

## Test in locale (prima di mettere online)

Apri `index.html` nel browser, compila il form, controlla che il lead appaia in Firestore (Console → Firestore → collezione `leads`).

Poi apri `dashboard.html`, fai login con le credenziali create al punto 3, e dovresti vedere il contatto nella tabella.

---

## Pubblicazione su Firebase Hosting (gratis)

1. Installa Firebase CLI:
   ```bash
   npm install -g firebase-tools
   ```
2. Dentro la cartella del progetto:
   ```bash
   firebase login
   firebase init hosting
   ```
   - Seleziona il progetto creato prima
   - Public directory: `.` (punto = cartella corrente)
   - Single-page app: **No**
   - Sovrascrivi `index.html`: **No**
3. Deploy:
   ```bash
   firebase deploy --only hosting
   ```
4. Ti restituisce un URL tipo `https://pn-autofficina.web.app` — quella è la landing live.
   La dashboard è su `https://pn-autofficina.web.app/dashboard.html`.

Dopo, per puntare un dominio personalizzato (es. `pnautofficina.it`):
Console Firebase → **Hosting** → **Aggiungi dominio personalizzato**.

---

## Cose da sapere

**La dashboard è "nascosta" ma non segreta.** Chiunque conosca l'URL `/dashboard.html` può vederla, ma senza credenziali non vede nessun contatto (lo blocca Firestore). Va bene così. Se vuoi nasconderla del tutto, puoi rinominarla in qualcosa di non indovinabile (es. `gestione-x9k2m.html`).

**La apiKey nel codice è pubblica e va bene così.** Non è una "chiave segreta": serve solo a identificare il progetto Firebase. La sicurezza vera è nelle **regole Firestore** e in **Auth**.

**Per aggiungere un secondo utente alla dashboard**, vai su Console → Authentication → Utenti → Aggiungi utente.

**Limiti gratuiti Firebase (piano Spark)** — più che sufficienti per un'officina:
- Firestore: 50.000 letture/giorno, 20.000 scritture/giorno, 1 GiB storage
- Auth: utenti illimitati con email/password
- Hosting: 10 GB storage, 360 MB/giorno traffico

**Backup:** dalla dashboard puoi esportare CSV in qualsiasi momento (bottone "Esporta CSV"). Tieni un backup periodico.

---

## Modifiche frequenti

**Cambiare la password della dashboard:** Console → Authentication → seleziona utente → menu `⋮` → Reimposta password (o cambia direttamente).

**Aggiungere un campo al form:** modifica `index.html` (sia il form HTML sia l'oggetto `data` nello script), aggiungilo a `dashboard.html` come colonna, aggiornare le regole `firestore.rules` per includere il nuovo campo in `hasOnly([...])`.

**Cambiare stati lead:** in `dashboard.html` cerca le opzioni `nuovo`/`contattato`/`chiuso` nel `<select>` e modifica.
