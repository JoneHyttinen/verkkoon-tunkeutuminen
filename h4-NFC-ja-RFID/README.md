# h4 - NFC ja RFID

---

## a) Tarkastele käytössäsi olevia RFID tuotteita

Omat RFID-tuotteet:

```
Maksukortti
Passi
iLOQ-avain
Roskiksen tagi
```

**Maksukortti (Lähimaksu, EMV)**

- Toimii 13,56 MHz:n taajudella (ISO 14443). Lukuetäisyys on käytännössä
muutamia senttejä.
- Lukija voi ilman PIN-koodia lukea ainakin kortin numeron ja voimassaoloajan, joskus
myös tapahtumahistorian, CVV-koodia ei voi lukea, eikä nimeä yleensä enää ole sirulla.
- Korttia ei voi kloonata toimivaksi, koska jokainen maksu allekirjoitetaan kertakäyttöisellä
kryptogrammilla.
- **Arvio:** Hyvin suojattu kloonausta vastaan. Kortin numeron urkinta onnistuu,
jos joku saa lukijan aivan kiinni lompakkoon.

**Passi**

- Myös 13,56 MHz (ICAO 9303). Sirulle on tallennettu henkilötiedot, kasvokuva
ja joskus sormenjäljet.
- Sirua ei voi lukea ilman avainta. Avain johdetaan passin tietosivun koneluettavasta MRZ-
rivistä (BAC/PACE -protokolla), joten lukijan pitää tietää passinumero, syntymäaika ja
voimassaoloaika. Sormenjäljet on suojattu vielä erikseen (EAC).
- **Arvio:** Erittäin hyvin suojattu. Suurin riski on, että joku kuvaa passin tietosivun,
ei RFID-urkinta.
- **Suojaustoimi:** Käytännöllisin olisi RFID-suojattu lompakko tai kotelo.

**iLOQ-avain (K5S, iLOQ S5-järjestelmä)**

- Avaimessa ei ole paristoa. S5-lukkosylinteri saa energiansa liikkeestä, joka syntyy kun avain
työnnetään lukkoon. Avaimen ja lukon välinen tiedonsiirto kulkee kosketuksen kautta (1-wire).
- Avaimessa on silti myös langaton liitäntä: siinä on NFC-siru, joka tukee RFID- ja NFC-
toimintoja. Sirussa on AES-256-salaus, ja avain toimii tiedonkuljettajana D2D-verkossa.
Käytännössä NFC:n kautta avaimen kulkuoikeudet päivitetään ja se toimii ovien NFC-
lukijoissa. Esimerkiksi N506i-lukijalla voi päivittää K5S-avainten kulkuoikeudet ja
aikaohjelmat.
- Langattomalla lukijalla avaimesta saa todennäköisesti selville ainakin sirun UID:n eli
tunnisteen. Ites kulkuoikeusdata on AES-salattua, joten avainten kloonaaminen on paljon
vaikeampaa kuin vanhoissa MIFARE Classic -tageissa.
- Mahdollinen heikkous ei ole avaimessa vaan järjestelmässä: iLOQ:n NFC-lukijat tukevat
myös MIFARE-RFID-tageja. Jos taloyhtiö on ottanut ne käyttöön, ovi voi aueta myös
heikommin suojatuilla tageilla.
- **Arvio:** Hyvin suojattu. Urkkija voi lähinnä tunnistaa avaimen yksilöllisen ID:n, jos pääsee
aivan sen viereen.

**Roskiksen tagi**

- Tyypillisesti matalataajuinen (LF) tunniste. Esimerkiksi norjalainen jätehuoltoyhtiö käyttää
astioihin kiinnitettyjä 125 kHz:n LF-RFID-tageja, joita jäteauton nostimeen asennetut
lukijat lukevat.
- Tagissa on vain yksilöllinen ID. ID:n avulla jäte voidaan yhdistää tiettyyn osoitteeseen, mutta
itse tagissa ei ole henkilötietoja.
- **Arvio:** Urkittavaa ei juuri ole. Kopioitu ID voisi teoriassa sotkea tyhjennysten kirjanpitoa
ja laskutusta.

**Yhteenveto ja suojatoimet**

- Suojaa tarvitsee lähinnä maksukortti ja mahdollinen MIFARE-tagi. Passi suojaa itse itsensä
ja roskiksen tagissa ei ole mitään arvokasta.

---

## b) APDU-komentojen rakenne

**Mikä APDU on?**

Älykorttien yhteydessä APDU (Application Protocol Data Unit) on viestiyksikkö, jolla kortinlukija
ja älykortti kommunikoivat. APDU:n rakenne on määritelty standardissa ISO/IEC 7816-4.
Langattomissa korteissa samat APDU:t kulkevat ISO 14443-4 protokollan (ISO-DEP) päällä.
Siksi maksukorttia voi puhutella samoilla komennoilla sekä sirun kosketuspinnan, että NFC:n kautta.

APDU-viestit kulkevat pareittain: lukija lähettää komennon (C-APDU) ja kortti vastaa (R-APDU).

**Komento-APDU (C-APDU)**

`| CLA | INS | P1 | P2 | Lc | Data | Le |`

| Kenttä | Pituus | Merkitys |
| --------------- | --------------- | --------------- |
| CLA | 1 tavu | Komentoluokka. `00` = ISO-standardin komento, `80` = valmistaja- tai sovelluskohtainen (esim. EMV), `0C` = suojattu viestintä (secure messaging) |
| INS | 1 tavu1 | Itse komento, esim `A4` = SELECT |
| P1,P2 | 1 + 1 tavua | Komennon parametrit |
| Lc | 0, 1 tai 3 tavua | Data-kentän pituus |
| Data | Lc tavua | Komennon mukana lähetettävä data |
| Le | 0-3 tavua | Kuinka monta tavua vastausdataa odotetaan |

CLA kertoo komentoluokan ja INS-tietyn komennon sen sisällä. P1 ja P2 tarkentavat komentoa. Lc
ilmoittaa data-kentän tavumäärän, joka voi olla enintään 255 tavua, ja Le vastauksen data-kentän
enimmäispituuden. Pidemmille viesteille on laajennettu muoto: kolmen tavun Lc, jonka
ensimmäisen tavun on oltava 0, ilmaisee datan pituuden väliltä 1-65 535. Le-kentässä arvo `00`
tarkoittaa "anna kaikki, enintään 256 tavua".

Kenttien läsnäolon mukaan komennot jaetaan neljään tapaukseen (case):

| Case | Rakenne | Esimerkki |
| --------------- | --------------- | --------------- |
| 1 | `CLA INS P1 P2` | Komento ilman dataa ja vastausdataa |
| 2 | `CLA INS P1 P2 Le` | Lukeminen, esim. READ RECORD |
| 3 | `CLA INS P1 P2 Lc Data` | Kirjoittaminen, esim. UPDATE BINARY |
| 4 | `CLA INS P1 P2 Lc Data Le` | Data menee kortille ja vastaus takaisin, esim. SELECT |

**Vastaus APDU (R-APDU)**

`| Data (valinnainen) | SW1 | SW2 |`

Kortti lähettää vastauksessa 0-65 536 tavua dataa ja kaksi pakollista tilatavua (SW1, SW2).
Yleisimpiä tilakoodeja:

| SW1 SW2   | Merkitys    |
|--------------- | --------------- |
| `90 00` | Onnistui   |
| `61 XX`   | Dataa on vielä XX tavua, hae se GET RESPONSE -komennolla   |
| `6C XX`   | Väärä Le, oikea pituus on XX   |
| `69 82`   | Turvaehto ei täyty (esim. autentikointi puuttuu)   |
| `6A 82`   | Tiedostoa tai sovellusta ei löydy   |
| `6D 00`   | Tuntematon INS   |
| `6E 00`   | Tuntematon CLA   |

**Yleisiä komentoja (INS)**

| INS | Komento | Käyttö |
| --------------- | --------------- | --------------- |
| `A4` | SELECT | Valitse sovellus (AID) tai tiedosto |
| `B0` | READ BINARY | Lue tiedostoa (passi) |
| `B2` | READ RECORD | Lue tietue (maksukortti) |
| `CA` | GET DATA | Hae yksittäinen tietoalkio |
| `84` | GET CHALLENGE | Pyydä kortilta satunnaisluku autentikointia varten |
| `82` | (MUTUAL/EXTERNAL) AUTHENTICATE | Autentikointi |
| `20` | VERIFY | PIN-koodin tarkistus |
| `C0` | GET RESPONSE | Hae loput vastausdatasta |
| `A8` | GET PROCESSING OPTIONS | EMV: aloita maksutapahtuma (CLA `80`) |
| `AE` | GENERATE AC | EMV: luo maksun kryptogrammi (CLA `80`) |

(Tähän tehtävään käytetty Anthropicin Claude Opus 5.5 tekoälymallia APDU-komentojen rakenteisiin tutustumiseen)

---

## c) Mielenkiintoinen RFID-hakkerointiuutinen

**Tausta**

MIFARE Classic on pitkään ollut tunnetusti heikko. Korttien käytöstä ei ole kuitenkaan päästy
eroon: korttien vaihtaminen uudempiin on monissa organisaatioissa ollut hidasta kustannusten ja
logistiikan takia, ja tästä on syntynyt kokonainen MIFARE Classic -yhteensopivien korttien
teollisuus, jonka suurimpia toimijoita on Shanghai Fudan Microelectronics.

Fudan julkaisi noin vuonna 2020 FM11RF08S-kortin, joka suunniteltiin kestämään kaikki
tunnetut pelkkään korttiin kohdistuvat hyökkäykset. Vastatoimen nimi on “static encrypted
nonce".

**Mitä löydettiin**

Quarkslabin tutkija Philippe Teuwen julkaisi elokuussa 2024 tutkimuksen, jossa hän kehitti uusia
hyökkäyksiä tätä suojattua korttia vastaan. Pikaisella fuzzauksella löytyi laitteistotason takaovi,
jonka kautta korttiin voi autentikoitua tuntemattomalla avaimella. Tutkija murtoi salaisen
avaimen ja totesi sen olevan sama kaikissa FM11RF08S-korteissa. Teuwenin mukaan takaoven
tunteva taho voi murtaa kortin kaikki käyttäjän asettamat avaimet, vaikka ne olisi diversifioitu,
kunhan pääsee käsiksi korttiin muutamaksi minuutiksi.

Sama ongelma löytyi vanhemmistakin korteista. Samanlainen takaovi oli myös edellisen
sukupolven FM11RF08-korteissa, mutta eri avaimella suojattuna. Tutkimuspaperin mukaan
vanhemmista korteista löytyi lisäksi takaoviavain, joka oli yhteinen useille valmistajille.

**Miksi tämä on merkittävää**

- **Kortteja on kaikkialla:** Haavoittuvia FM11RF08- ja FM11RF08S-kortteja käytetään
hotelleissa Yhdysvalloissa, Euroopassa, Kiinassa ja Intiassa.
- **Käyttäjä ei tiedä omistavansa niitä:** Moni ei luultavasti tiedä, että toimittajalta ostetut
MIFARE Classic -kortit ovatkin Fudanin FM11RF08- tai FM11RF08S-kortteja, eikä niiden
käyttö rajoitu Kiinan markkinoihin.
- **Toimitusketjuriski:** Tutkimuksen mukaan hyökkäykset voisi tehdä välittömästi taho, joka
pystyy toimitusketjuhyökkäykseen. Valmistaja, joka tuntee takaoven, voisi siis kloonata
kortin hetkessä.
- **Takaovi eroaa vanhoista heikkouksista:** MIFARE Classicin aiemmat ongelmat johtuivat
suunnitteluvirheistä. Tämä on tiettävästi tarkoituksella piirille rakennettu ominaisuus.
- **Testaustyökalut ovat julkisia:** Quarkslab lisäsi työkalunsa ja menetelmänsä Proxmark3:n
repositorioon, jotta tietoturva-ammattilaiset voivat testata järjestelmiään.

Uutinen osoittaa, että RFID-järjestelmän turvallisuus riippuu sen heikoimmasta tunnisteesta.
Oma iLOQ K5S -avaimeni on AES-256-suojattu, mutta jos taloyhtiön lukijat hyväksyvät myös
MIFARE-tagit, ja tagi sattuu olemaan Fudanin halpa klooni, oveen pääsee käsiksi kopioimalla sen
kortin. Hyväksyttyjen korttien ja tagien vaihtaminen moderneihin, esimerkiksi MIFARE DESFire
EV2/EV3:een, on ainoa todellinen korjaus.

---

## Lähteet

- [Radio-frequency identification (Wikipedia)](https://en.wikipedia.org/wiki/Radio-frequency_identification)
- [iLOQ S5-avain tuotesivu](https://www.iloq.com/en-gb/catalogue/k5s-5/)
- [NF506i RFID Reader](https://support.iloq.com/en-au/iloq-manager/5-series/user-guide/iloq-online-system-and-devices/iloq-online-system/n50x-rfid-readers/n506i-rfid-reader/)
- [RFID Journal 2013](https://www.rfidjournal.com/news/norwegian-trash-collector-uses-rfid-to-clean-up-its-operations/83745/)
- [Smart card application protocol data unit. (Wikipedia)](https://en.wikipedia.org/wiki/Smart_card_application_protocol_data_unit)
- [MIFARE Classic: exposing the static encrypted nonce variant and a few hardware backdoors](https://blog.quarkslab.com/mifare-classic-static-encrypted-nonce-and-backdoors.html)
- [Backdoor in Mifare Smart Cards Could Open Doors Around the World](https://www.infosecurity-magazine.com/news/backdoor-mifare-smart-cards-open/)

Anthropic. Claude Opus 5.5 tekoälymalli
