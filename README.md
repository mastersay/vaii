# Water Drink Reminder

Jakub Prusek

5ZYI32

## Stručný opis projektu
Water Drink Reminder je multiplatformová aplikácia (web, Android, iOS, desktop) na sledovanie pitného režimu. Používateľ zaznamenáva vypité nápoje, pričom každý nápoj má vlastný hydratačný koeficient (napr. káva hydratuje menej ako voda). Aplikácia z pohlavia, hmotnosti a úrovne aktivity vypočíta odporúčaný denný príjem tekutín, pripomína pitie notifikáciami a zobrazuje štatistiky. Aplikácia funguje aj offline bez účtu, dáta sa ukladajú do lokálnej databázy.

Prihlásený používateľ si môže zapnúť synchronizáciu, vďaka ktorej má dáta na novom zariadení bez ručného exportu a importu. Môže sa spájať do skupín (rodina, priatelia) a zapájať sa do pretekov a udalostí s rebríčkom a odznakmi. 

## Technológie
- **Klient:** Kotlin Multiplatform + Compose Multiplatform (Wasm/JS, Android, iOS, JVM desktop), lokálna databáza SQLDelight (SQLite), komunikácia so serverom cez Ktor client
- **Server:** NestJS (TypeScript), PostgreSQL, REST API + WebSocket (živé rebríčky, upozornenie na zmeny pri synchronizácii)
- **Autentifikácia:** Clerk (e-mail + heslo, Google). Server overuje tokeny Clerku a používateľov zrkadlí do vlastnej tabuľky.

## Role v projekte
- **Návštevník** (bez prihlásenia) – plnohodnotná offline aplikácia nad lokálnou databázou (záznamy, štatistiky, nastavenia, notifikácie, export/import). Zo serverovej časti vidí iba aktuálne verejné udalosti a ich rebríčky. Nemôže sa zapojiť do udalostí ani skupín, nezískava odznaky a nesynchronizuje dáta.
- **Registrovaný používateľ** – všetko ako návštevník, navyše synchronizácia dát, verejný profil s avatarom, členstvo v skupinách, účasť v udalostiach a získavanie odznakov.
- **Vlastník skupiny** – rola v rámci konkrétnej skupiny, získava ju zakladateľ skupiny. Spravuje skupinu, pozýva a odoberá členov a vytvára súkromné preteky skupiny.
- **Administrátor** – spravuje verejné udalosti, definície odznakov a používateľov a moderuje skupiny.

## Prípady použitia podľa rolí
### Návštevník
- zaznamená nápoj rýchlym pridaním alebo z katalógu, upraví alebo zmaže záznam
- nastavenie profilu
- prezeranie štatistiky a histórie záznamov
- spravuje katalóg nápojov, notifikácie, tému a jednotky
- exportuje a importuje dáta do súboru
- prezerá aktuálne verejné udalosti a ich rebríčky
- zaregistruje sa alebo prihlási

### Registrovaný používateľ
- zapne synchronizáciu a na novom zariadení si stiahne svoje dáta
- upraví zobrazované meno a nahrá avatar
- vytvorí skupinu, prijme alebo odmietne pozvánku, opustí skupinu
- prihlási sa do verejnej udalosti alebo preteku svojej skupiny, sleduje živý rebríček, odhlási sa
- prezerá svoje odznaky a odznaky členov svojich skupín

### Vlastník skupiny
- upraví názov, popis,... skupiny, zmaže skupinu
- pozve používateľa, zruší pozvánku, odoberie člena
- vytvorí, upraví a zmaže súkromné udalosti/preteky skupiny

### Administrátor
- vytvorí, upraví a zmaže verejnú udalosť (obdobie, spôsob hodnotenia, podmienka na odznak, odznak)
- spravuje definície odznakov za osobné míľniky
- mení roly používateľov, blokuje účty a maže nevhodné skupiny

## Plánované entity
Serverová databáza (PostgreSQL). Synchronizované entity (označené *sync*) majú rovnakú schému ako lokálna SQLite databáza, doplnenú o `user_id`, UUID identifikátor, `updated_at` a `deleted_at` (zmazanie sa synchronizuje ako príznak).

| Entita | Účel | Hlavné atribúty |
|---|---|---|
| User | používateľ zrkadlený z Clerku (nezapočítava sa medzi entity) | clerk_id, username, display_name, avatar_path, role (user/admin), sync_enabled |
| UserProfile | parametre na výpočet cieľa (*sync*) | gender, weight_kg, lifestyle, weather_type |
| AppSettings, NotificationSettings | nastavenia aplikácie a pripomienok (*sync*) | theme, units, week_start / enabled, interval_minutes, start_hour, end_hour |
| DrinkType | katalóg nápojov používateľa (*sync*) | name, hydration_impact, default_amount_ml, icon, color, is_active, sort_order |
| DrinkEntry | záznam vypitého nápoja (*sync*) | drink_type_id, amount_ml, logged_at |
| DailyGoal | denný cieľ a jeho úpravy (*sync*) | date, goal_ml, is_adjustment |
| Group | skupina (rodina, priatelia) | name, description, avatar_path |
| GroupMember | členstvo v skupine | group_id, user_id, role (owner/member), joined_at |
| GroupInvitation | pozvánka do skupiny | group_id, invited_user_id, invited_by_id, status (pending/accepted/declined) |
| Event | verejná udalosť alebo súkromný pretek skupiny | title, description, starts_at, ends_at, scoring (days_met/streak), completion_threshold, group_id (prázdne = verejná), badge_id, created_by_id |
| EventParticipant | účasť v udalosti a priebežné skóre | event_id, user_id, joined_at, days_met, best_streak, rank |
| Badge | definícia odznaku | name, description, icon, type (event/placement/milestone), criteria |
| UserBadge | získaný odznak | user_id, badge_id, event_id, awarded_at |

Skóre v udalostiach počíta server zo synchronizovaných záznamov, preto účasť v udalosti vyžaduje zapnutú synchronizáciu.

## Vzťahy medzi entitami
- **User 1:1** UserProfile, AppSettings, NotificationSettings
- **User 1:N** DrinkType, DrinkEntry, DailyGoal
- **DrinkType 1:N** DrinkEntry
- **User M:N Group** cez GroupMember, ktorý nesie aj rolu v skupine
- **Group 1:N** GroupInvitation, **User 1:N** GroupInvitation (ako pozvaný aj ako pozývajúci)
- **Group 1:N Event** (súkromné preteky, verejné udalosti nemajú skupinu)
- **User M:N Event** cez EventParticipant, **User 1:N Event** ako autor
- **Badge 1:N Event** (odznak za dokončenie udalosti)
- **User M:N Badge** cez UserBadge, voliteľne s väzbou na udalosť, za ktorú bol odznak udelený

## Hlavné stránky aplikácie
1. **Úvodné nastavenie** – prvé spustenie, profil, výpočet denného cieľa, voliteľné prihlásenie
2. **Domov** – prstenec s postupom dňa rozdelený podľa farieb nápojov, ukazovateľ očakávaného množstva, rýchle pridanie nápoja, záznamy dňa
3. **Štatistiky** – grafy za deň, týždeň, mesiac a rok, plnenie cieľa, rozdelenie podľa nápojov
4. **Záznamy** – celá história nápojov rozdelená podľa dní s úpravou a mazaním
5. **Nastavenia** – profil a cieľ, notifikácie, katalóg nápojov, vzhľad, jednotky, export/import, účet a synchronizácia
6. **Prihlásenie a registrácia** – cez Clerk
7. **Udalosti** – aktuálne, nadchádzajúce a skončené verejné udalosti a preteky mojich skupín s filtrovaním
8. **Detail udalosti** – pravidlá, obdobie, odznak, živý rebríček, prihlásenie a odhlásenie
9. **Skupiny** – moje skupiny, prijaté pozvánky, vytvorenie skupiny
10. **Detail skupiny** – členovia s avatarmi a odznakmi, preteky skupiny, správa členov, pozvánok a pretekov pre vlastníka
11. **Profil a odznaky** – zobrazované meno, avatar, získané odznaky
12. **Administrácia** – správa verejných udalostí, odznakov a používateľov

## Rozdelenie funkcionality
### Základné
- offline sledovanie pitného režimu: záznam nápojov, výpočet cieľa, štatistiky, história, notifikácie, katalóg nápojov, export/import
- registrácia a prihlásenie cez Clerk, roly používateľ, vlastník skupiny a administrátor, autorizácia na serveri
- voliteľná synchronizácia celej lokálnej databázy medzi zariadeniami
- profil s nahraním avatara
- skupiny: vytvorenie, úprava vrátane obrázka, zmazanie, pozvánky podľa používateľského mena s prijatím alebo odmietnutím, odobratie člena, opustenie skupiny
- udalosti: verejné (administrátor) aj súkromné preteky skupiny (vlastník), prihlásenie a odhlásenie, hodnotenie podľa dní so splneným cieľom alebo najdlhšej série, živý rebríček cez WebSocket
- odznaky za dokončenie udalosti, umiestnenie v prvej trojke a osobné míľniky (napr. 7- a 30-dňová séria, prvých 100 l)
- administrácia udalostí, odznakov a používateľov
- kontrola vstupov na klientovi aj na serveri

### Rozširujúce
- zdieľanie histórie a štatistík v rámci skupiny s nastavením súkromia
- spoločné ciele skupiny (napr. „každý člen si splní pitný režim počas jedného týždeňa“)
