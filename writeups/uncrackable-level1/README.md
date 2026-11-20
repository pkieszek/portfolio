# PATSEC toolkit

Darmowy zestaw do bug bounty na Maca M1/M2. Zero platnych narzedzi. Twoj Burp Community robi manual, te skrypty robia automat obok.

## Pliki

| plik | co robi |
|---|---|
| `patsec-setup.sh` | instaluje caly toolchain (raz) |
| `patsec-lib.sh` | wspolna biblioteka (poprawne dopasowanie scope). MUSI lezec obok reszty |
| `patsec-scan.sh` | menu: recon domeny, scan URL, porty, crawl, XSS, mobile |
| `patsec-monitor.sh` | codzienny monitor nowych assetow (TWOJA przewaga) |
| `patsec-hunt.sh` | moduly secret sauce: takeover, 403 bypass, CORS, XSS, JS intel |
| `patsec-parallel.sh` | skanuje wiele programow naraz, rownolegle, z limitami |

**Wazne:** trzymaj wszystkie pliki `patsec-*` w jednym katalogu. Skrypty laduja `patsec-lib.sh` z sasiedztwa i bez niego nie ruszą.

## Start (3 kroki)

```bash
chmod +x patsec-*.sh
./patsec-setup.sh          # instalacja, 5-15 min
# otworz NOWE okno terminala
./patsec-scan.sh           # menu, opcja 1 = pelny recon
```

Wszystkie wyniki laduja w `~/patsec-results/`.

---

## Strategia (przeczytaj zanim odpalisz)

Prawda ktorej nikt poczatkujacym nie mowi: **lepszy skaner nie da Ci kasy, bo nuclei ma kazdy.** Wszyscy skanuja te same stare assety tymi samymi szablonami. Przewage masz z trzech rzeczy i dokladnie pod to jest ten zestaw:

**1. Byc pierwszym na nowym assecie.** Firmy codziennie wypuszczaja nowe subdomeny, staging, feature flagi. Kto pierwszy je przeskanuje, bierze buga. To robi `patsec-monitor.sh` na cronie. Ustaw raz, chodzi samo, alert leci na telefon jak cos nowego wyskoczy.

**2. Logika biznesowa, nie skanery.** Byles na sandboksie bookingu Kiwi. To jest ZLOTO i tego zaden skaner nie zlapie: manipulacja cena, IDOR na cudzej rezerwacji, przejscie platnosci, powtorzenie requestu. Tu wygrywasz glowa + Burp Community (Repeater, podmiana requestow). Automat Ci tylko mapuje teren, bugi logiczne znajdujesz Ty.

**3. Dobor programu.** O tym nizej.

### Nisza vs codzienny automat

Masz dwie drogi i obie sa dobre, mozesz laczyc:

- **Nisza (malo ludzi):** swiezo dodane programy, male firmy, programy VDP bez kasy ktore potem przechodza na platne, zakresy ktore wymagaja wysilku (mobile, API, poddomeny glebokie). Mniej konkurencji = Twoj automat czesciej jest pierwszy. Szukaj na HackerOne/Intigriti sortujac po "newest", i bierz programy z szerokim scope (`*.domena.com`).
- **Duzy program codziennie (Kiwi itp.):** duzo ludzi, ale duzy scope i ciagle zmiany. Tu wygrywasz monitorem, nie jednorazowym skanem. `patsec-monitor.sh kiwi-domena` na cronie i czekasz az wyskoczy nowy host zanim inni go zobacza.

Mobile to osobna nisza i dobra: **mniej hunterow umie ruszyc apki.** MobSF (opcja 6 w scan) rozklada .apk/.ipa, wyciaga klucze API, zahardkodowane sekrety, slabe ustawienia. Czesto leza tam szybkie winy ktorych web-ludzie nie dotykaja. Skoro wahales sie miedzy Kiwi a mobile, to wlasnie mobile moze byc Twoja mniej zatloczona droga.

### Gdzie szukac programow (darmowe)

- HackerOne `hackerone.com/directory` i `hackerone.com/opportunities` (sortuj newest / launched)
- Intigriti (duzo europejskich, mniej US-tloku)
- YesWeHack (sporo nisz, EU)
- Bugcrowd
- Listy publiczne: `github.com/projectdiscovery/public-bugbounty-programs` (da sie ciagnac programatycznie do monitora)
- `chaos.projectdiscovery.io` gotowe listy subdomen scope dla wielu programow

---

## Codzienny monitor na cronie

To jest ta czesc co "puszczasz codziennie zeby cos wylapac".

```bash
crontab -e
# codziennie 9:00, jeden wpis na cel:
0 9 * * *  /bin/bash /PELNA/SCIEZKA/patsec-monitor.sh example.com >> ~/patsec-monitor/example.com/log.txt 2>&1
```

Alert na telefon (opcjonalnie) przez webhook Discorda:
```bash
echo 'export PATSEC_WEBHOOK="https://discord.com/api/webhooks/TWOJ_URL"' >> ~/.zshrc
```
Zrob prywatny serwer Discord, kanal, Edit Channel > Integrations > Webhook, skopiuj URL. Alerty leca na komorke za darmo.

---

## Workflow ktory realnie placi

1. `patsec-monitor.sh` na cronie na 3-5 programach. Chodzi samo.
2. Jak wyskoczy nowy host albo cos w JS, siadasz.
3. `patsec-hunt.sh` moduly A-E na tym hoscie (takeover, bypass, CORS, XSS, JS intel).
4. Co ciekawe recznie w **Burp Community** (Repeater). Tu robisz logike i IDOR.
5. Mobile: `patsec-scan.sh` opcja 6, wrzucasz apke do MobSF, czytasz raport.

Burp Community wystarcza na 90% manuala: proxy, Repeater, Decoder, podmiana requestow. Brakuje aktywnego skanera i Intrudera bez throttlingu, ale to Ci zastepuje nuclei + ffuf + dalfox z tego zestawu. Nie potrzebujesz Pro na start.

---

## Dyscyplina scope (wazne)

Wszystkie skrypty maja guard: nie puszczaja celu ktorego nie ma w `~/.patsec-scope`, i zawsze odrzucaja to co jest w `~/.patsec-oos`. Skanujesz **tylko** to co program jawnie dopuszcza. Out-of-scope = zero kasy i realne ryzyko. To higiena zawodowa ktora Cie chroni.

Format wpisu w `~/.patsec-scope` oraz `~/.patsec-oos`:

```
example.com        # apex + wszystkie subdomeny *.example.com
*.acme.io          # to samo, apex + subdomeny
=only.this.host    # TYLKO ten jeden host, bez subdomen
# linia komentarza
```

Dopasowanie jest po granicy domeny, nie po tekscie. Czyli `example.com` w scope NIE przepusci `notexample.com` ani `example.com.evil.com`. Jak program ma out-of-scope (np. `blog.example.com`), wrzuc to do `~/.patsec-oos` i guard to odrzuci nawet gdy wpada pod wildcard.

---

## Aktualizacja

W `patsec-scan.sh` opcja 7, albo:
```bash
pdtm -ua                   # narzedzia
nuclei -update-templates   # szablony (nowe CVE co tydzien)
```
Aktualizuj szablony czesto. Nowy szablon CVE w dniu wyjscia + Twoj monitor = najlepsze okno na szybki win.
