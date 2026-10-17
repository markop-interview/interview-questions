# Go Interview Playbook

# 40 čestih Go pitanja za intervju — sa odgovorima

**Backend i Senior Backend Engineer**

*Osnove → Slices i Maps → Concurrency → Internals runtime-a → Produkcioni obrasci*

**Dodatno:** 12 coding zadataka, 5 zamki i 10 scenarija za Senior pozicije.

---

Go intervjui se ne padaju zbog sintakse.

Padaju se zbog `concurrency`, memorije i razumevanja šta se dešava kada stvari krenu po zlu.

U nastavku je 40 pitanja koja se zaista pojavljuju na backend intervjuima, zajedno sa odgovorima.

Pripremajte se ovim redosledom: osnove → `slices` i `maps` → `concurrency` → interni mehanizmi → produkcija.

## Q1. Semantika vrednosti (*value semantics*) naspram semantike pokazivača (*pointer semantics*) u Go-u?

Svako dodeljivanje pravi kopiju. Kod skalarnih tipova — `int`, `string`, `struct` — kopira se sama vrednost. Kod `slices`, `maps`, `channels` i pokazivača kopira se zaglavlje ili pokazivač (kod `slice`-a: pokazivač plus `len` i `cap`), tako da obe kopije upućuju na iste osnovne podatke. To je suština deljenja referenci (*aliasing*) u jednoj rečenici. Ako vam je potrebna duboka kopija (*deep copy*), morate je napraviti sami.

> ⚠️ Napomena: string i struct nisu skalarni tipovi. Kopiranje string vrednosti kopira zaglavlje (pokazivač + dužina) i deli iste bajtove, što je bezbedno jer su stringovi nepromenljivi. Kopiranje struct-a kopira sva polja, a polja tipa slice, map, pokazivač ili interface i dalje dele iste podatke.}}$

## Q2. Koje su nulte vrednosti (*zero values*) u Go-u i zašto su važne?

Svaki tip ima upotrebljivu nultu vrednost: `0`, `""`, `false`, `nil`, odnosno `struct` u kome su sva polja već na nultim vrednostima. Zato su deklaracije `var mu sync.Mutex` i `var m map[string]int` bezbedne. Međutim, `nil map` može da se čita i iz nje mogu da se brišu elementi, ali upisivanje u nju izaziva `panic`. `nil channel` blokira zauvek u oba smera. Upravo ove dve situacije često izazivaju probleme u produkciji.

$\color{red}{\textsf{⚠️ Napomena: Nije svaka nulta vrednost upotrebljiva: var m map[string]int je bezbedno samo za čitanje, len, range i delete, a upis izaziva panic. var mu sync.Mutex jeste odmah upotrebljiv. Dodatno: close nad nil channel-om izaziva panic.}}$

## Q3. Objasnite `defer`, `panic` i `recover`.

`defer` se izvršava pri izlasku iz funkcije, po LIFO redosledu. Argumenti se evaluiraju odmah, a ne u trenutku izvršavanja odloženog poziva. `panic` odmotava stek i pritom izvršava odložene funkcije; `recover` funkcioniše samo kada se pozove direktno unutar odložene funkcije i samo u istoj `goroutine`. Koristite `panic` za greške programera, a ne za očekivane greške. Za sve što pozivalac može da obradi, vraćajte `error`.

## Q4. Kada treba koristiti `pointer receiver`?

Osnovno pravilo je doslednost. Ako bilo kojoj metodi nekog tipa treba `pointer receiver`, koristite `pointer receiver` za sve metode tog tipa. Pokazivači su neophodni i kada metoda mora da izmeni `receiver`, ili kada je `struct` dovoljno veliki da je trošak njegovog kopiranja pri svakom pozivu merljiv. Važna zamka: tip vrednosti zadovoljava interfejs samo ako sve njegove metode imaju `value receiver`.

$\color{red}{\textsf{⚠️ Napomena: Preciznije: vrednost tipa T zadovoljava interfejs samo ako sve metode koje taj interfejs zahteva imaju value receiver (method set od T sadrži samo njih). Ostale metode tipa mogu imati pointer receiver. Method set od *T sadrži metode sa oba tipa receiver-a.}}$

## Q5. Šta je ugrađivanje tipa (*struct embedding*) i čemu služi?

Ugrađivanjem tipa njegova polja i metode postaju dostupni preko spoljnog `struct`-a. To je korisno za ponovno korišćenje ponašanja bez nasleđivanja. Zamka: ugrađivanjem `interface`-a spoljni `struct` ga naizgled automatski „implementira“, a poziv metode koju ste zaboravili da nadjačate može u runtime-u da izazove `panic` zbog `nil` vrednosti. Ako odnos nije stvarni odnos tipa „jeste“ (*is-a*), prednost dajte eksplicitnoj delegaciji.

## Q6. Kako funkcionišu greške (*errors*) i koja je razlika između `errors.Is` i `errors.As`?

Greške su vrednosti. `fmt.Errorf("...: %w", err)` obavija grešku (*wrap*) i čuva lanac grešaka. `errors.Is(err, target)` proverava jednakost duž lanca — koristite ga za sentinel greške, kao što je `io.EOF`. `errors.As(err, &target)` prolazi kroz lanac i pronalazi grešku određenog tipa — koristite ga kada su vam potrebna polja, na primer kod `*net.OpError`. Pravilo: proveravajte pomoću `Is`, granajte logiku pomoću `As`, a greške obavijajte pri svakom prelasku granice apstrakcije.

## Q7. Kada su imenovane povratne vrednosti (*named return values*) zaista korisne?

U kratkim funkcijama u kojima odložena funkcija (`defer`) mora da izmeni rezultat, na primer `func f() (err error)`. U svim drugim slučajevima one umanjuju čitljivost. Česta greška je oslanjanje na goli `return` da vrati vrednosti koje je `defer` promenio — to funkcioniše samo sa imenovanim rezultatima.

$\color{red}{\textsf{⚠️ Napomena: defer menja rezultat kad god su rezultati imenovani, i uz goli return i uz return izraz (izraz se prvo upiše u imenovani rezultat, pa se izvrše odložene funkcije). Goli return nije uslov. Imenovani rezultati su korisni i kao dokumentacija kada funkcija vraća više vrednosti istog tipa, npr. (lat, lng float64).}}$

## Q8. Čemu služi `iota`?

`iota` je brojač konstanti koji se uvećava unutar svakog `const` bloka, a ne u svakom redu. Uobičajen obrazac je `type Level int`, pa zatim `const (Low Level = iota; Medium; High)`, uz metodu `String()` ili niz za pretragu. Ako ubacite vrednost u sredinu, sve naredne vrednosti se prenumerišu. Za trajno skladištenje ili wire formate, definišite cele brojeve eksplicitno.

$\color{red}{\textsf{⚠️ Napomena: Netačno: iota se resetuje na 0 na početku svake const deklaracije i uvećava se za svaki ConstSpec, tj. za svaki red u bloku, bez obzira da li se iota u tom redu koristi. Više konstanti u istom redu dobija istu vrednost iota.}}$

## Q9. Šta se promenilo u vezi sa hvatanjem promenljivih petlje (*loop variable capture*) u Go 1.22?

Promenljive deklarisane u petlji, uključujući promenljivu `range`, sada se kreiraju za svaku iteraciju. Klasična greška u kojoj sve `goroutine` ispisuju istu vrednost uklonjena je u verziji 1.22 i novijim. Kod napisan za starije verzije i dalje je ispravan, jer prosleđivanje promenljive kao argumenta funkcije i dalje funkcioniše. Ipak, proverite kod koji se namerno oslanjao na zajedničku promenljivu.

$\color{red}{\textsf{⚠️ Napomena: Nova semantika važi samo kada go direktiva u go.mod (ili build tag fajla) navodi go 1.22 ili noviju. Ako go.mod kaže go 1.21, i sa Go 1.26 kompajlerom važi staro ponašanje sa zajedničkom promenljivom. Odnosi se na promenljive iz range i iz init dela for petlje sa tri dela.}}$

## Q10. Kako interfejsi funkcionišu ispod haube?

Vrednost `interface`-a sastoji se od dve mašinske reči: pokazivača na tip i pokazivača na podatke. Skup metoda (*method set*) izračunava se statički, tokom kompajliranja. Nema troška `reflection`-a osim ako ga sami ne pozovete. Interfejse definišite na strani potrošača, a ne proizvođača — mali interfejs sa jednom metodom na mestu poziva bolji je od širokog interfejsa izvezenog iz paketa.

$\color{red}{\textsf{⚠️ Napomena: Kod praznog interface-a (any) prva reč je pokazivač na tip, a kod interface-a sa metodama pokazivač na itab (tip + tabela metoda). itab za konverzije u runtime-u (npr. type assertion na interfejs) računa se dinamički i kešira. Poziv preko interfejsa jeste indirektan poziv: obično sprečava inlining i može izazvati alokaciju na heap-u kada se vrednost smesti u interface.}}$

## Q11. Koja je razlika između `len` i `cap` kod `slice`-a?

`len` je broj elemenata kojima možete pristupiti indeksiranjem, a `cap` govori dokle `slice` može da se proširi pre nego što Go ponovo alocira memoriju. `make([]int, 3, 10)` daje tri elementa i prostor da se doda još sedam elemenata pomoću `append` bez kopiranja. Tako izbegavate ponovljeno proširivanje i troškove kopiranja u kritičnoj petlji (*hot loop*).

## Q12. Zašto `append` ponekad menja originalni `slice`?

Zato što `append` upisuje u zajednički osnovni niz (*backing array*) ako postoji slobodan kapacitet. Izraz `b := a[:2]; b = append(b, 99)` upisuje u niz na koji pokazuje `a`, pa `a` sada ispisuje `[1 2 99]`. Ako se premaši `cap`, Go alocira novi niz. Ovo je jedan od najčešćih izvora grešaka u produkcionom Go kodu i razlog zašto treba klonirati `slice` pre nego što ga prosledite drugom delu programa.

## Q13. Kada Go ponovo alocira `slice`?

Kada bi `len` premašio `cap`. Go povećava kapacitet približno geometrijski — udvostručuje ga kod manjih `slice`-ova, a kod velikih ga povećava za oko 1,25 puta — ali tačni faktori su detalj implementacije na koji se ne treba oslanjati. Ako znate veličinu unapred, unapred alocirajte memoriju pomoću `make([]T, 0, n)`.

## Q14. Da li je kopiranje `struct`-a isto što i duboka kopija (*deep copy*)?

Ne. Kopiraju se polja, ali svako polje tipa `slice`, `map` ili pokazivač i dalje upućuje na iste osnovne podatke. Duboke kopije moraju se praviti eksplicitno. Ako `struct` sadrži mutex, njegovim kopiranjem kopira se i stanje zaključavanja — nikada nemojte kopirati `struct` koji sadrži `sync.Mutex`; prosleđujte pokazivač.

## Q15. Šta se dešava kada pročitate nepostojeći ključ u `map`-i?

Dobijate nultu vrednost odgovarajućeg tipa. Izraz `v := m["nope"]` vraća `0`, `""`, `false` ili `nil`, u zavisnosti od tipa. Da biste razlikovali slučaj „ključ ne postoji“ od slučaja „ključ postoji, a vrednost je nulta“, koristite oblik `v, ok := m["nope"]`. Brisanje se vrši pomoću `delete(m, k)`; ako ključ ne postoji, operacija ne radi ništa.

## Q16. Da li je redosled iteracije kroz `map`-u garantovan?

Ne. Redosled je namerno nasumičan. Ako vam je potreban deterministički izlaz, prikupite ključeve u `slice` i sortirajte ih. Menjanje `map`-e tokom iteracije nije bezbedno za oslanjanje na određeno ponašanje: novi ključevi mogu, ali ne moraju, biti posećeni.

$\color{red}{\textsf{⚠️ Napomena: Specifikacija kaže da redosled nije definisan i ne garantuje se. Runtime ga dodatno nasumično bira, ali nije uniformno slučajan. Menjanje map-e tokom range iteracije u istoj goroutine je dozvoljeno i bezbedno: obrisani neposećeni ključevi neće biti posećeni, a novi mogu ali ne moraju. Brisanje tokom iteracije je uobičajen i ispravan obrazac.}}$

## Q17. Zašto se za `slice`, `string` i `map` kaže da zauzimaju „tri mašinske reči“?

Svaki od njih ima malo zaglavlje: pokazivač i dva cela broja. Njihovo prosleđivanje je jeftino, a tipovi se prosleđuju po vrednosti. Trošak je skriven — svaki `append` koji proširi `slice`, svaki unos u `map` i svako nadovezivanje `string` vrednosti mogu da izazovu alokaciju na heap-u. Zato se pri svakoj analizi performansi pominju `escape analysis` i unapredna alokacija (*preallocation*).

$\color{red}{\textsf{⚠️ Napomena: Netačno: samo slice ima tri mašinske reči (pokazivač, len, cap; 24 bajta na 64-bit). string ima dve (pokazivač, len; 16 bajtova), a map je jedan pokazivač na runtime strukturu (8 bajtova). Provereno pomoću unsafe.Sizeof.}}$

## Q18. Šta je `escape analysis`?

Kompajler odlučuje da li će promenljiva biti smeštena na stek (*stack*) ili heap, na osnovu toga da li pokazivač na nju nadživi funkciju. Ako vratite `&localVar`, promenljiva „izbegava“ stek i prelazi na heap. To možete videti komandom `go build -gcflags='-m'`. Vraćanje vrednosti umesto pokazivača obično omogućava da podatak ostane na steku.

## Q19. Koja je razlika između `goroutine` i OS niti (*OS thread*)?

`goroutine` ima mali stek kojim upravlja Go runtime, a raspoređuje se na OS niti. Počinje sa stekom veličine nekoliko kilobajta i raste po potrebi. Nitima upravlja operativni sistem, a njihov trošak uključuje kernel stek i vreme promene konteksta (*context switch*). Pokretanje `goroutine` je jeftino, pa pravo pitanje nije „možemo li da pokrenemo još?“ već „koliko njih istovremeno smemo da imamo u toku?“.

## Q20. Ne baferovani naspram baferovanih `channel`-a — kada koristiti koji?

Ne baferovani `channel` predstavlja tačku sinhronizacije: pošiljalac se blokira dok primalac ne bude spreman, čime se obezbeđuje direktna predaja. Baferovani `channel` razdvaja proizvođača i potrošača i ublažava nalete posla — birate koliko stavki sme istovremeno da bude u toku. Veličina bafera predstavlja politiku `backpressure`, a ne magičan broj.

## Q21. Ko zatvara `channel` i šta se dešava ako ga zatvorite dva puta?

`channel` zatvara pošiljalac, nikada primalac. Zatvaranje već zatvorenog `channel`-a izaziva `panic`, kao i slanje podataka kroz zatvoren `channel`. Primaoci mogu da otkriju zatvaranje pomoću `v, ok := <-ch`. Iteracija `range` nad `channel`-om završava se kada se on zatvori. Vlasništvo nad zatvaranjem treba jasno da se vidi iz potpisa funkcije.

## Q22. Šta radi `select` sa `default` granom?

Omogućava neblokirajuću operaciju: ako nijedan `case` nije spreman, izvršava se `default`. Dodavanjem slučaja sa `time.After` dobijate timeout. Tako se implementiraju pokušaj slanja bez blokiranja (*try-to-send*) i pražnjenje posla pri gašenju (*drain-on-shutdown*). `nil channel` unutar `select`-a blokira zauvek, što je idiomatski način da se određeni `case` privremeno isključi.

$\color{red}{\textsf{⚠️ Napomena: Timeout pomoću time.After radi samo u select-u BEZ default grane; ako default postoji, on se izvršava odmah i timeout nikada ne okine. nil channel u select-u nikada nije spreman, pa se taj case preskače; select blokira zauvek samo ako nema drugih spremnih case-ova ni default grane.}}$

## Q23. Kada koristiti `sync.WaitGroup`, a kada `errgroup`?

`WaitGroup` samo broji završene zadatke — koristite ga kada treba sačekati da se sve `goroutine` završe, a obrada grešaka je jednostavna. `errgroup.Group` obezbeđuje otkazivanje nakon prve greške i `context`. U produkciji je `errgroup` često bolji izbor, jer „sačekaj sve“ bez „prekini ostale ako nešto ne uspe“ obično predstavlja grešku u dizajnu.

$\color{red}{\textsf{⚠️ Napomena: errgroup je paket golang.org/x/sync/errgroup, ne standardna biblioteka. Otkazivanje posle prve greške postoji samo uz errgroup.WithContext; Wait i dalje čeka sve goroutine i vraća prvu grešku. SetLimit ograničava broj istovremenih goroutine-a. Od Go 1.25 postoji i sync.WaitGroup.Go.}}$

## Q24. `Mutex` naspram `RWMutex` naspram atomskih operacija — kako izabrati?

Podrazumevano izaberite običan `Mutex`. `RWMutex` se isplati samo kada je čitanja znatno više nego upisa i kritična sekcija je duga. Atomske operacije koristite za pojedinačne brojače i zastavice. Kopiranje mutex-a je greška. Mutex štiti invarijantu, a ne samo jedno polje, zato dokumentujte šta štiti.

## Q25. Šta izaziva deadlock u Go-u i kako ga otkloniti?

Ciklusi čekanja između `goroutine`-a mogu izazvati deadlock. Kada su sve `goroutine` blokirane na `channel`-ima, runtime može da prijavi `panic` porukom `all goroutines are asleep - deadlock!`. Ta poruka ispisuje sve stekove, što obično olakšava uočavanje ciklusa. Manje očigledna varijanta je sporo curenje resursa, pa ograničite blokirajuće operacije pomoću `select`-a uz timeout ili `context`.

$\color{red}{\textsf{⚠️ Napomena: Netačno: to nije panic nego fatal error (fatal error: all goroutines are asleep - deadlock!), koji se ne može uhvatiti pomoću recover. Runtime ga prijavljuje samo kada su SVE goroutine blokirane (na channel-ima, mutex-ima itd.); delimični deadlock se tako ne otkriva. Od Go 1.26 postoji eksperimentalni goroutineleak profil (GOEXPERIMENT=goroutineleakprofile) koji otkriva i takve slučajeve.}}$

## Q26. Razlika između `data race` i `race condition`?

`data race` je nedefinisano ponašanje izazvano nesinhronizovanim konkurentnim pristupom istoj memoriji. `race condition` je logička greška kod koje rezultat zavisi od vremenskog rasporeda izvršavanja. Opcija `-race` otkriva samo prvi tip. Klasičan primer: dve `goroutine` uvećavaju običan `int` — možda neće biti upozorenja, ali rezultat može biti pogrešan.

$\color{red}{\textsf{⚠️ Napomena: Primer je pogrešan: dve goroutine koje uvećavaju običan int bez sinhronizacije prave upravo data race, i -race ga prijavljuje (provereno). Primer race condition-a bez data race-a: provera pa akcija (npr. proveri da li je stanje dovoljno, pa skini iznos) gde su čitanje i upis svaki zasebno zaštićeni mutex-om ili atomikom, ali ne zajedno.}}$

$\color{red}{\textsf{Go memory model data race ne naziva nedefinisanim ponašanjem u smislu C-a, ali rezultat može biti nekonzistentan, a kod višerečnih vrednosti (interface, slice, string) i oštećenje memorije.}}$

## Q27. Kako treba koristiti `context`?

Uvek ga prosleđujte kao prvi parametar. On prenosi otkazivanje, rokove (*deadlines*) i vrednosti. Nikada nemojte čuvati `context` u `struct`-u, jer time životni vek objekta vezujete za nepovezan zahtev. U petljama proveravajte `ctx.Err()`. Za svaki odlazni poziv koji može da se zaglavi koristite `context.WithTimeout`.

## Q28. Kako ograničiti `concurrency` umesto da pokrenete sve odjednom?

Koristite baferovani `channel` kao semafor ili `worker pool` fiksne veličine. Za 10.000 zapisa ne želite 10.000 `goroutine`-a i 10.000 otvorenih konekcija — želite, na primer, 50. Ograničenja konkurentnosti štite bazu podataka, a ne samo CPU.

## Q29. Kako napraviti `worker pool`?

Koristite `jobs channel` kapaciteta N, `results channel`, fiksan broj worker-a i `WaitGroup` ili `errgroup`. Odlučite ko zatvara `jobs channel`, ko zatvara `results` i kako se propagiraju greške i otkazivanje. Ograničen broj poslova obezbeđuje `backpressure`, što je i glavna svrha ovog dizajna.

## Q30. Kako dolazi do curenja `goroutine`-a (*goroutine leak*)?

`goroutine` može zauvek da ostane blokirana pri slanju na `channel` sa kog niko ne čita, pri prijemu sa `channel`-a na koji niko ne šalje, tokom mrežnog poziva bez roka ili zbog `time.Tick` koji nikada nije zaustavljen. Sve ove `goroutine` mogu da nadžive zahtev koji ih je pokrenuo. Curenje možete otkriti pomoću `runtime.NumGoroutine()` i `goroutine profile`; rešava se korišćenjem `context`-a i eksplicitnim upravljanjem gašenjem.

$\color{red}{\textsf{⚠️ Napomena: Zastarelo: od Go 1.23 Timer i Ticker na koje se više ne upućuje mogu da se počiste i bez Stop, pa time.Tick više ne curi sam po sebi. Curi goroutine koja zauvek čeka na tick channel-u, a Stop ne zatvara channel, pa je ne odblokira. Rešenje je select sa ctx.Done() ili done channel-om.}}$

## Q31. Kako funkcioniše garbage collection u Go-u?

Go koristi konkurentni `mark-and-sweep` garbage collector. On je precizan, a ne konzervativan. Faza `mark` radi konkurentno sa vašim `goroutine`-ama, koristeći `tri-colour abstraction` i `write barriers`, pa su `stop-the-world` pauze kratke — u praksi kraće od milisekunde — ali proces označavanja troši CPU, kao i sama stopa alokacije. Parametri za podešavanje uključuju `GOGC`, `GOMEMLIMIT`, `SetMemoryLimit` i `SetGCPercent`. Od Go 1.26, `Green Tea` collector je podrazumevano uključen i smanjuje trošak označavanja kod velikih heap-ova. Pratite stopu alokacije, a ne samo veličinu heap-a.

$\color{red}{\textsf{⚠️ Napomena: Green Tea (podrazumevan od Go 1.26) ubrzava označavanje i skeniranje malih objekata zahvaljujući boljoj lokalnosti i skalabilnosti na više jezgara; release notes navode oko 10 do 40 procenata manji trošak GC-a. Nije posebno vezan za velike heap-ove. Može se isključiti sa GOEXPERIMENT=nogreenteagc.}}$

## Q32. Objasnite G, M i P.

`G` je `goroutine`, `M` je OS nit, a `P` je logički procesor koji omogućava izvršavanje `G`-a. `M` može da izvršava Go kod samo dok poseduje `P`, a broj `P`-ova jednak je vrednosti `GOMAXPROCS`. Scheduler održava lokalni red spremnih zadataka za svaki `P` i preuzima posao iz drugih redova kada se njegov isprazni. Zato je `GOMAXPROCS` parametar paralelizma, a ne broj `goroutine`-a. `goroutine` blokirane u sistemskim pozivima odvajaju svoj `P`, kako bi ostale mogle da nastave sa radom.

## Q33. Šta je tipizirani `nil` (*typed nil*) i zašto poređenje sa `== nil` može da zavara?

Vrednost `interface`-a sadrži dve reči: tip i vrednost. Poređenje sa `nil` daje `true` samo kada su oba dela `nil`. Zato je `x == nil` netačno u sledećem primeru:

```go
var p *int
var x any = p
x == nil // false
```

To često pravi probleme u obradi grešaka i konstruktorima koji vraćaju pokazivač tipa `nil`. Rešenja su vraćanje običnog `nil` za `error`, provera pomoću `reflection`-a ili preuređivanje koda tako da se u `interface` nikada ne upiše `nil` vrednost.

$\color{red}{\textsf{⚠️ Napomena: Isečak se ne kompajlira: x == nil kao samostalna naredba daje grešku (value is not used). Treba npr. fmt.Println(x == nil), kao u zamci 2.}}$

## Q34. Za šta su generici (*generics*) korisni u Go-u?

Za ponovo upotrebljive strukture podataka i algoritme sa proverom tipova — na primer, tipizirani red (*queue*), tipizirani `LRU`, generički `merge` ili `reduce` — u situacijama kada biste inače duplirali kod za svaki tip. `Constraints` prihvataju tipove koji zadovoljavaju određeno ponašanje, na primer `~int | ~int64`. Generici nisu zamena za interfejse pri povezivanju servisa (*service wiring*), a trošak instanciranja može biti važan u kritičnim putanjama (*hot paths*).

$\color{red}{\textsf{⚠️ Napomena: Unija tipova (npr. tilde int ili tilde int64) ograničava skup dozvoljenih tipova po osnovnom tipu, a ne po ponašanju; ponašanje opisuju metode u constraint-u. Trošak u hot path-u ne dolazi od instanciranja nego od implementacije sa GC shape-ovima i rečnicima: pozivi metoda preko type parametra su indirektni i mogu da spreče inlining.}}$

## Q35. Kako funkcionišu Go moduli?

`go.mod` definiše putanju modula i potrebne verzije, a `go.sum` beleži hash vrednosti koje je `proxy` proverio. Verzije prate `semantic import versioning` — verzije `v2` i novije koriste putanju koja sadrži `/v2`. `go mod tidy` uklanja nepotrebne zavisnosti, `go.work` upravlja repozitorijumima sa više modula, a `GOPRIVATE` preskače javni `proxy` za privatne module.

$\color{red}{\textsf{⚠️ Napomena: go.sum beleži očekivane kriptografske hash-eve sadržaja modula i go.mod fajlova; proveravaju se prema checksum bazi (sum.golang.org), ne prema proxy-ju. go mod tidy i dodaje zavisnosti koje nedostaju. go.work definiše workspace sa više modula za lokalni razvoj. GOPRIVATE preskače i proxy i checksum bazu.}}$

## Q36. Kako implementirati `dependency injection` bez framework-a?

Prosleđujte zavisnosti kao parametre konstruktora i držite interfejse malim, definisanim na strani potrošača. Na primer:

```go
func NewService(repo OrderRepo, cache Cache, log *slog.Logger) *Service
```

Nisu potrebni `reflection` ni kontejner, a testovi mogu da dobiju `fake` zavisnost jednom linijom koda. Povežite zavisnosti u `main`. Ako vam je odmah potreban framework za `dependency injection`, moguće je da su interfejsi smešteni u pogrešan paket.

## Q37. Kako profilisati Go servis?

Izložite `net/http/pprof` i ograničite pristup tom endpoint-u. `CPU profile` otkriva gde se troši vreme; `heap profile` sa `-inuse_space` pokazuje memoriju koja je i dalje zauzeta, dok `-alloc_space` pokazuje stopu alokacije — ona određuje trošak GC-a. `goroutine profile` služi za otkrivanje curenja, a `block` i `mutex profiles` za analizu čekanja i konkurencije oko zaključavanja. `go tool trace` pomaže pri analizi latencije scheduler-a. Najpre napravite početno merenje (*baseline*), pa upoređujte profile, a ne samo brojke.

$\color{red}{\textsf{⚠️ Napomena: alloc\_space prikazuje ukupne bajtove alocirane od starta programa, ne stopu; stopu dobijate poređenjem dva profila (-diff\_base) ili iz runtime/metrics. block i mutex profili su prazni dok ih ne uključite pomoću runtime.SetBlockProfileRate i runtime.SetMutexProfileFraction.}}$

## Q38. Kako testirati Go kod?

Koristite paket `testing`, testove zasnovane na tabelama (*table-driven tests*) i podtestove pomoću `t.Run`. Prednost dajte `fake` implementacijama umesto `mock`-ovima — mali ručno napisan `fake` često je bolji od generisanog `mock`-a koji morate da održavate. Pažljivo koristite `t.Parallel()`, pokrećite konkurentne testove uz `-race`, pravite benchmark za kod osetljiv na alokacije i prosledite sat kao zavisnost umesto da čekate pomoću `sleep`. Česta korisna komanda je:

```sh
go test -race -benchmem ./...
```

$\color{red}{\textsf{⚠️ Napomena: Komanda je pogrešna: -benchmem nema efekta bez -bench, pa se benchmark-ovi uopšte ne pokreću. Treba: go test -race ./... za testove, i odvojeno go test -bench=. -benchmem ./... (benchmark-e ne merite uz -race, jer usporava kod).}}$

## Q39. Kako izgleda `worker pool` u produkciji?

Koristi se ograničen `jobs channel`, fiksan broj worker-a, `results channel`, `errgroup` sa `context`, `WaitGroup` za pražnjenje reda i putanja za gašenje: prestanite da prihvatate nove poslove, zatvorite `jobs`, dozvolite da se posao koji je već u toku završi i zatvorite `results`. Svaki worker proverava `ctx.Done()` između jedinica posla, a svaki spoljni poziv mora imati rok. Zatim izaberite garanciju isporuke: *at-least-once* uz `idempotent` handler ili *at-most-once*.

## Q40. Kako elegantno ugasiti HTTP server (*graceful shutdown*)?

Osluškujte `SIGINT` i `SIGTERM` preko posebnog `channel`-a, a zatim pozovite `srv.Shutdown(ctx)` uz timeout. `Shutdown` prestaje da prihvata nove konekcije i čeka da se zahtevi koji su već u toku završe. Postavite rok koji ste spremni da čekate i pozovite `srv.Close()` ako istekne. Redosled je važan: prvo uklonite servis iz service discovery-ja ili load balancer-a, zatim ugasite HTTP server, potom pozadinske worker-e i na kraju bazu podataka.

---

# 12 coding pitanja

## 1. Obrnite string (*Reverse a string*)

Iterirajte kroz `rune` vrednosti, a ne kroz bajtove:

```go
for _, r := range s {
    out = append(out, r)
}
```

Testirajte sa emoji znakom; obrtanje bajtova daje neispravan rezultat.

$\color{red}{\textsf{⚠️ Napomena: Kod ne obrće string, samo kopira rune istim redosledom (i out nije deklarisan). Ispravno: r := []rune(s); slices.Reverse(r); return string(r). Napomena: obrtanje runa i dalje kvari složene emoji (zastave, ZWJ sekvence) i kombinovane znakove; za njih treba rad po graphemskim klasterima.}}$

## 2. Uklonite duplikate iz `slice`-a

Jedan prolaz uz `map`, dodavanjem elementa pri prvom pojavljivanju, ima složenost `O(n)`. Ako redosled nije važan, sortiranje u praksi često može biti brže.

## 3. Pronađite prvi znak koji se ne ponavlja

U jednom prolazu prebrojte učestalost znakova, a zatim ponovo prođite kroz ulaz i pronađite prvi znak čija je učestalost `1`. Koristite bajtove ako je ulaz ASCII, a `rune` vrednosti ako nije.

## 4. Implementirajte `stack` i `queue`

`Stack` može biti `slice` uz `append` i `s[len(s)-1]`. `Queue` može biti `slice` sa indeksom početka ili `channel`. Zatim objasnite zašto postoji kružni bafer (*ring buffer*) kada se red nikada ne smanjuje.

## 5. Implementirajte `LRU cache`

Koristite `container/list` za dvostruko povezanu listu i `map` koja povezuje ključ sa elementom liste. `Get` i `Put` premeštaju element na početak. Operacije imaju složenost `O(1)`, a pri izbacivanju elementa (*eviction*) morate da ga uklonite iz liste.

$\color{red}{\textsf{⚠️ Napomena: Pri izbacivanju element morate ukloniti i iz liste i iz map-e; zato element liste treba da čuva i ključ, a ne samo vrednost. Get premešta element na početak samo kada ključ postoji.}}$

## 6. Implementirajte `worker pool`

Pogledajte Q29. Ograničen broj poslova, fiksan broj worker-a, `results channel`, `WaitGroup`, otkazivanje i ograničen red za `backpressure`.

## 7. Napravite `rate limiter`

Koristite `token bucket` sa mutex-om ili biblioteku `golang.org/x/time/rate`. Ograničenja po korisniku zahtevaju `bucket` za svaki ključ uz izbacivanje neaktivnih stavki (*eviction*); u suprotnom dolazi do curenja memorije.

## 8. Implementirajte `thread-safe counter`

Za jedan brojač koristite `atomic.AddInt64`. Ako štitite više polja koja moraju da se menjaju zajedno, koristite `Mutex`. Upravo je ta razlika suština pitanja.

## 9. Spojite rezultate iz više `goroutine`-a

Koristite `WaitGroup` uz `slice` zaštićen mutex-om ili baferovani `results channel` i posebnu `goroutine` koja ga zatvara. Razmotrite redosled rezultata i objasnite zašto je neuređen izlaz često sasvim prihvatljiv.

## 10. Implementirajte ograničeni red poslova (*bounded job queue*)

Ograničenje može da bude baferovani `channel` ili `semaphore channel` koji se zauzima pre pokretanja `goroutine`-a. Kada se red napuni, odbijte posao, blokirajte pozivaoca ili odbacite posao — navedite koju opciju birate, jer je to odluka proizvoda.

## 11. Napišite omotač sa timeout-om (*timeout wrapper*)

Primite `context`, a u pozivaocu koristite `select` nad `results channel`-om i `time.After`. Imajte na umu da ćete ostaviti `goroutine` koja curi ako se unutrašnji poziv nikada ne vrati — zato i sam poziv mora da ima rok.

$\color{red}{\textsf{⚠️ Napomena: Ako primate context, select treba da čeka ctx.Done() (uz context.WithTimeout i defer cancel()), a ne time.After. results channel mora biti baferovan (kapacitet 1); u suprotnom unutrašnja goroutine curi čim timeout istekne, čak i kada se poziv kasnije vrati.}}$

## 12. Implementirajte mehanizam ponovnog pokušaja (*retry mechanism*)

Ograničite broj pokušaja, koristite eksponencijalno čekanje (*exponential backoff*) uz `jitter`, ponavljajte samo `idempotent` operacije, prekinite kada se `ctx` otkaže i poštujte `Retry-After`. Nemojte dodavati retry logiku u biblioteku ispod transportnog sloja, jer ćete dobiti više slojeva ponovljenih pokušaja.

---

# 5 zamki

## 1. Šta nije u redu sa ovim kodom?

```go
for _, v := range values {
    go func() {
        fmt.Println(v)
    }()
}
```

U Go 1.22 i novijim verzijama — ništa. Promenljiva `range` kreira se za svaku iteraciju. Pre verzije 1.22 sve `goroutine` su delile istu promenljivu i ispisivale poslednju vrednost. Prosleđivanje `v` kao argumenta i dalje je eksplicitan i prenosiv obrazac.

$\color{red}{\textsf{⚠️ Napomena: Nije baš ništa: main ne čeka goroutine, pa program može da se završi pre ispisa, a redosled ispisa nije definisan. Treba WaitGroup (npr. wg.Go od Go 1.25). Odgovor o 1.22 važi samo kada je go direktiva u go.mod 1.22 ili novija.}}$

## 2. Šta će ovaj kod ispisati?

```go
var p *int
var x any = p
fmt.Println(x == nil)
```

`false`. `interface` sadrži dinamički tip `*int` i `nil` vrednost, pa sam `interface` nije `nil`.

## 3. Zašto `append` može da vas iznenadi?

```go
a := []int{1, 2, 3}
b := a[:2]
b = append(b, 99)
```

Ispisuje `[1 2 99]`. `b` deli osnovni niz sa `a` i ima slobodan kapacitet. Ako se kapacitet premaši, Go alocira novi niz.

## 4. Može li `goroutine` da curi i nakon što se handler završi?

Može. To se dešava ako je blokirana pri slanju ili prijemu preko `channel`-a, tokom mrežnog poziva bez roka ili na `ticker`-u koji nije zaustavljen. Rešenje su otkazivanje i eksplicitno upravljanje životnim ciklusom.

$\color{red}{\textsf{⚠️ Napomena: Ticker koji nije zaustavljen od Go 1.23 nije uzrok curenja (GC ga počisti kad nema referenci), a Stop ne zatvara channel, pa ne odblokira goroutine koja čeka na ticker.C. Goroutine curi zato što zauvek čeka; rešenje je select sa ctx.Done().}}$

## 5. Da li je redosled iteracije kroz `map`-u garantovan?

Ne. Nikada se ne oslanjajte na njega. Sortirajte ključeve kada vam je potreban deterministički redosled.

---

# 10 pitanja za Senior pozicije

## 1. Deset `goroutine`-a ažurira istu `map`-u. Kako to učiniti bezbednim?

Koristite jedan mutex ili `sync.Map` ako je pristup zaista pretežno čitalački. Bezbednost proverite konkurentnim testom uz `-race`. Zatim razmotrite da li vam je zaključavanje uopšte potrebno: možete podeliti `map`-u na segmente (*sharding*) ili dodeliti vlasništvo jednoj `goroutine` i komunicirati preko `channel`-a.

## 2. Latencija Go API-ja raste pod opterećenjem. Kako biste istražili uzrok?

Počnite od obrasca rasta: da li je uzrok CPU, zaključavanje ili nizvodna zavisnost (*downstream dependency*)? Napravite `CPU profile` — ako je problem CPU, videćete ga tamo. Ako nije, pregledajte `goroutine profile` i potražite hiljade blokiranih `goroutine`-a, a zatim `block` i `mutex profiles` zbog čekanja i konkurencije oko zaključavanja. Potom upotrebite `go tool trace` za analizu latencije scheduler-a. Uporedite ponašanje pod opterećenjem i u mirovanju. Tek tada formirajte hipotezu. Česta greška je nagađanje pre profilisanja.

## 3. Upotreba heap memorije stalno raste iako se GC izvršava. Šta biste proverili?

Najpre utvrdite da li raste živi heap (*live heap*) ili samo RSS memorija koju operativni sistem još nije povratio. Uporedite `HeapAlloc` i `HeapInuse` sa `HeapReleased`, a zatim proverite broj `goroutine`-a — one koje cure zadržavaju sve objekte na koje upućuju. Napravite `inuse_space` profil i pregledajte objekte koji zadržavaju najviše memorije. Uobičajeni uzroci su curenje `goroutine`-a, neograničeni keševi, `slice`-ovi koji se nikada ne skraćuju i rastuća `map`-a koja se pretežno čita, a za koju bi mogao biti prikladan `sync.Map`.

$\color{red}{\textsf{⚠️ Napomena: sync.Map ne rešava rast memorije. Uzrok je obično to što map-a ne vraća memoriju posle delete (ni Swiss table implementacija od Go 1.24 se ne smanjuje); rešenje je povremeno praviti novu map-u ili ograničiti veličinu keša. Još jedan čest uzrok: mali sub-slice ili substring koji drži ceo veliki osnovni niz (rešenje: slices.Clone, strings.Clone).}}$

## 4. Worker se sruši dok obrađuje posao. Kako sprečiti gubitak posla?

Učinite handler `idempotent` i potvrdite prijem posla (*acknowledge*) tek nakon što se sporedni efekat uspešno potvrdi (*commit*). Dodajte `visibility timeout` za ponovnu isporuku, ograničene retry pokušaje sa `backoff`-om i `dead-letter queue` uz alarm. Garancija *at-least-once* uz idempotentnost realističan je izbor. Oporavite se od `panic`-a zasebno za svaki posao, tako da jedan neispravan payload ne ugasi worker.

## 5. Klijent prekine vezu dok handler čeka rezultat upita ka bazi. Šta treba da se dogodi?

`context` zahteva se otkazuje, upit vraća `context.Canceled`, a handler prekida posao koji više nikome nije potreban. Prosledite `ctx` upitu i koristite `select` nad `ctx.Done()` oko operacija koje ne podržavaju `context`. Obavljajte samo potrebno čišćenje, kao što su beleženje događaja (*logging*) i oslobađanje rezervacije. Kada je sve pravilno povezano, otkazivanje funkcioniše prirodno; bez toga možete ostaviti konekciju i neobjašnjivu `goroutine` koja curi.

## 6. Kako implementirati ograničenje broja zahteva po korisniku (*per-user rate limiter*) kroz više instanci servisa?

Lokalni `limiter` na svakoj instanci daje približno globalno ograničenje ako limit podelite brojem instanci, ali je neprecizan pri automatskom skaliranju (*autoscaling*). Za stvarno zajedničko ograničenje, stanje držite izvan procesa: koristite `Redis token bucket` sa `Lua` skriptom kako bi operacija čitanje–izmena–upis bila atomska, ili poseban servis za ograničavanje stope zahteva. Alternativno, primenite ograničenje na mrežnoj ivici (*edge*). Zatim objasnite ponašanje pri kvaru: ako `Redis` nije dostupan, da li sistem propušta zahteve (*fail open*) ili ih blokira (*fail closed*)?

## 7. Kako ugasiti servis bez gubitka zahteva koji su već u toku?

Najpre uklonite servis iz service discovery-ja ili load balancer-a i sačekajte da se promena proširi. Zatim sprovedite graceful HTTP shutdown sa rokom, prestanite da prihvatate pozadinske poslove i sačekajte da se postojeći završe (*drain*), ispraznite baferovane metrike i na kraju zatvorite bazu podataka. Gašenje treba da bude `idempotent`. Testirajte ga tako što ćete u staging okruženju poslati `SIGTERM` dok je servis pod opterećenjem.

## 8. Više `goroutine`-a ažurira isti zapis. Kako izbeći nekonzistentno stanje?

Prenesite ograničenje u bazu podataka: koristite optimističku konkurentnost (*optimistic concurrency*) sa kolonom `version` i upitom `UPDATE ... WHERE version = ?`, ili `SELECT FOR UPDATE` za operacije čitanje–izmena–upis. U Go-u možete usmeriti sve upise kroz jednu vlasničku `goroutine` za svaki ključ ako je stanje u memoriji. Dve `goroutine` sa odvojenim kopijama istog zapisa u memoriji pre ili kasnije će se razići.

## 9. Kako testirati konkurentni kod i otkriti race uslove?

Pokrećite `go test -race` pri svakom CI prolazu, koristite `t.Parallel()` za podtestove, gde je moguće koristite determinističku sinhronizaciju i za nestabilne testove dodajte stresno ponavljanje sa `-count=100`. Eksplicitno testirajte otkazivanje i gašenje, jer se tu često pojavljuju curenja. Proverite da li postoji curenje poređenjem `runtime.NumGoroutine()` pre i posle testa. Pokrećite i dugotrajni test (*soak test*) sa `-race` — detektor prijavljuje samo trke koje je zaista uočio.

## 10. Kako napraviti posao koji se može ponavljati (*retryable job*) tako da bude `idempotent`?

Dodelite poslu stabilan ključ idempotentnosti (*idempotency key*), najčešće poslovni ID, i nametnite jedinstvenost na odredištu: jedinstveni indeks uz `INSERT` ili `dedupe` tabelu sa `TTL`. Proveru obavite pre sporednog efekta, a ne posle njega. Prednost dajte prirodno idempotentnim operacijama: `SET` umesto inkrementiranja, `upsert` i direktno postavljanje apsolutne vrednosti umesto obrasca čitanje–izmena–upis. Sve što se ne može učiniti idempotentnim zahteva transakciju koja zajedno beleži efekat i ID posla.

---

**40 pitanja. Jedan redosled pripreme koji funkcioniše.**

Ako zapamtite samo jednu stvar: prvo detaljno proučite `concurrency` i interne mehanizme runtime-a, a zatim produkcione scenarije. To je ono što razlikuje iskusnog Go backend inženjera od nekoga ko samo poznaje sintaksu.

Koje pitanje vam najčešće postavljaju? Prikupljam pitanja koja se stalno ponavljaju.

**Tagovi:** `#Golang` `#GoLang` `#BackendEngineering` `#Concurrency` `#DistributedSystems` `#SystemDesign` `#DevInterview` `#InterviewPreparation` `#SoftwareArchitecture` `#GoDeveloper` `#CodingInterview`
