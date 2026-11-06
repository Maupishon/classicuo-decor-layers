# Dekoracyjne layery (fork ClassicUO, baza `b13d3fe82`, gałąź `decor-layers`)

## Numeracja (bajt layera w tiledata i w pakietach)

| Wartość | Nazwa | Rysowanie |
|---|---|---|
| 0x20 | TattooRArm | przed Shirt |
| 0x21 | TattooLArm | przed Shirt |
| 0x22 | TattooRLeg | przed Shirt |
| 0x23 | TattooLLeg | przed Shirt |
| 0x24 | TattooTorso | przed Shirt |
| 0x25 | TattooHead | przed Shirt |
| 0x26 | Belt2 ("pas2") | zaraz po Waist |
| 0x27 | Head2 (kaptur, nad Helmet) | zaraz po Helmet |

0x1E i 0x1F pominięte celowo: w ServUO 0x1E to `Reserved_x1E`, a 0x1F to `SecureTrade`
(`SecureTradeContainer` ma `Layer = Layer.SecureTrade`, `Server/Items/SecureTradeContainer.cs:17`).
W świecie tatuaże w wierszu kierunku 3 stoją po Cloak, w pozostałych na początku listy.

## Serwer ServUO: minimalna zmiana (opis; w tym repo nie ma kodu serwera)

1. `Server/Item.cs`, `enum Layer : byte` (linie ~183-200): po `SecureTrade = 0x1F,` jest blok `/* Reserved_x20 ... */`
   (zakomentowany). Dodać przed nim:
   ```csharp
   TattooRArm = 0x20,
   TattooLArm = 0x21,
   TattooRLeg = 0x22,
   TattooLLeg = 0x23,
   TattooTorso = 0x24,
   TattooHead = 0x25,
   Belt2 = 0x26,
   Head2 = 0x27,
   ```
   W komentowanym bloku usunąć `Reserved_x20..x26`, żeby nie dublować (w komentarzu nie przeszkadzają, ale mylą).
2. Pakiety: nic. `Packets.cs` (~3755-3775) wysyła `(byte)item.Layer` i dedupuje po `m_DupedLayers[256]`, więc 0x20-0x26 przejdą.
3. Zapis/odczyt: nic. `Item.Serialize` zapisuje layer jako `byte` (`m_Layer = (Layer)reader.ReadByte()`, `Item.cs:3462/3675/3814`).
4. Zakładanie: `Item.CanEquip` (`Item.cs:1774`) wymaga `Layer != Invalid` i pustego slotu. `EquipReq` (0x13) nie sprawdza zakresu layera.
5. Do sprawdzenia przed wdrożeniem (nie sprawdzone): skrypty z pętlami po `Layer` lub rzutowaniem `(Layer)i` (np. `LevelItemManager`, `Possess`, `Mannequin` w skryptach Nelderim). `BulletinBoards` używa `<= Layer.Mount`, więc nowych layerów nie obejmie.

### Wzorzec itemu: tatuaż prawej ręki
`Scripts/Nelderim/Items/Decor/TattooRArm.cs` (propozycja ścieżki; wzór stylu: `DaggerBelt.cs`):
```csharp
namespace Server.Items
{
    public class TattooRArm : Item
    {
        // 0x0000 = PLACEHOLDER: wpisz ItemID nadany przez pipeline (tiledata: layer 0x20, animId, wearable).
        [Constructable]
        public TattooRArm()
            : base(0x0000)
        {
            Weight = 0.0;
            Layer = Layer.TattooRArm;
            LootType = LootType.Blessed;   // nie wypada przy śmierci
            Name = "tatuaż prawej ręki";
        }

        public TattooRArm(Serial serial)
            : base(serial)
        {
        }

        public override void Serialize(GenericWriter writer)
        {
            base.Serialize(writer);
            writer.Write(0); // version
        }

        public override void Deserialize(GenericReader reader)
        {
            base.Deserialize(reader);
            int version = reader.ReadInt();
        }
    }
}
```
Czysty decor: brak `BaseClothing`/`BaseArmor`, więc zero wpływu na AR i statystyki.

## Przykład end-to-end: jeden testowy tatuaż

Użyj KOPII klienta, nie działającego. Komendy pipeline'u Nelderim (`nelderim.py ...`) z jego dokumentacji; ścieżki podmień.

1. Wolne ID itemu i animacji: `python nelderim.py search --client <KOPIA> --item "<nazwa>"`; wolne body HUMAN/EQUIPMENT jest też w `FREE_SLOTS.md` klienta.
2. Animacja: narysuj tatuaż jako ubranie (PNG RGBA) i zbuduj `.vd` 35 akcji x 5 kierunków dla ciała 400 jak zwykłe ubranie (`build_item.py` / `atlas_to_vd.py`, `vdtool info` = typ 2, 35x5). Wstrzyknięcie do anim5: `nelderim.py inject` / `wire` (dry-run, potem `--apply`).
3. Gump paperdolla: męski = AnimID + 50000, damski = + 10000 (`make_gump.py` / `nelderim.py gump3d`, zapis do `gumpartLegacyMUL.uop` przez `uop_gump_patch.py`).
4. Ikona art (`art.mul`/UOP wg klienta).
5. Tiledata: dla ItemID ustaw `layer = 0x20`, `animId = <body>`, flagę Wearable. NIEZWERYFIKOWANE: czy `nelderim_patch` potrafi ustawić pole layer. Jeśli nie, UOFiddler (Items, pole Layer). Klient wysyła przy zakładaniu layer z tiledata (`GameActions.cs:559`), więc tiledata musi mieć 0x20.
6. Equipconv.def (jeśli potrzebne): `400 <ItemID-animId> <conv> <gump> <hue>`.
7. ServUO: enum Layer + klasa `TattooRArm` z `ItemID` z kroku 1; restart serwera; `[add TattooRArm`, założyć.
8. Klient: skopiuj do kopii klienta `ClassicUO.exe`, `ClassicUO.Assets.dll`, `ClassicUO.IO.dll`, `ClassicUO.Renderer.dll`, `ClassicUO.Utility.dll` z folderu, do którego zbudowałeś klienta (patrz README).

## Budowanie

Wymagane: .NET SDK (u nas 8/9/10), podmoduły FNA i MP3Sharp. Jeśli `git://` jest zablokowane, pobierz zagnieżdżone podmoduły FNA przez https:

```bash
git submodule update --init external/MP3Sharp
git -C external/FNA -c url."https://github.com/".insteadOf=git://github.com/ submodule update --init --recursive
dotnet build src/ClassicUO.Client/ClassicUO.Client.csproj -c Release -o build-out
```

Uruchomienie z danymi klienta UO (PowerShell):

```powershell
.\build-out\ClassicUO.exe -uopath "C:\sciezka\do\klienta" -clientversion 7.0.95.0 -ip 127.0.0.1 -port 2593
```

Do działającego klienta kopiuj tylko `ClassicUO.exe` i `ClassicUO.{Assets,IO,Renderer,Utility}.dll`.

## Merge do własnego klienta (np. innego forka 1.x)

Zmiany to 6 commitów na `b13d3fe82` (gałąź `decor-layers`). Najprościej:

```bash
git remote add decor https://github.com/Maupishon/classicuo-decor-layers.git
git fetch decor
git merge decor/decor-layers        # albo: git cherry-pick b13d3fe82..decor/decor-layers
```

Miejsca, w których mogą być konflikty (to jedyne pliki kodu, które ruszamy):
- `src/ClassicUO.Client/Game/Data/Layers.cs`: wartości enuma 0x20-0x27 na końcu.
- `src/ClassicUO.Client/Game/Constants.cs`: `USED_LAYER_COUNT` (31) i `BAD_CONTAINER_LAYERS` (dopisane wpisy do indeksu 0x27).
- `src/ClassicUO.Client/Game/Data/LayerOrder.cs`: w każdym z 8 wierszy tatuaże przed `Shirt` (w wierszu 3 po `Cloak`), `Belt2` po `Waist`, `Head2` po `Helmet`.
- `src/ClassicUO.Client/Game/UI/Controls/PaperDollInteractable.cs`: te same wpisy w `_layerOrder` i `_layerOrder_quiver_fix`.
- `src/ClassicUO.Client/Network/PacketHandlers.cs`: warunek odświeżenia paperdolla w `EquipItem` (`Layer.Mount` lub `>= Layer.TattooRArm`).

Jeśli Twój klient ma własne layery, po scaleniu sprawdź, że `USED_LAYER_COUNT` = liczba pozycji w każdym wierszu `LayerOrder` i obu tablicach paperdolla, a każdy nowy layer jest w `BAD_CONTAINER_LAYERS`.

## Ograniczenia klienta 1.x (sprawdzone w kodzie)
- Animacja ekwipunku jest rysowana tylko dla `AnimID` (z tiledata) mniejszego niż **2048**: `MobileView.DrawInternal` odrzuca `id >= Animations.MaxAnimationCount`, a ta tablica startuje z 2048 i rośnie dopiero wewnątrz `GetAnimationFrames`, czyli za późno. Dotyczy też zwłok i wierzchowców. Zdjęcie limitu to powiększenie `_dataIndex` w `ClassicUO.Renderer/Animations/Animation.cs` (nie zrobione w tym repo).
- Rysowane są tylko layery z `LayerOrder.UsedLayers`; layer spoza listy zostaje przyjęty z pakietu, ale jest niewidoczny.
- Jeden przedmiot na layer (`Entity.FindItemByLayer`).
- Nowe layery nie są klikalne w świecie (pixel-check w `MobileView` kończy się na `Layer.Mount`).

### Co sprawdzić w grze
- Pod koszulą: załóż Shirt i tatuaż; tatuaż ma być widoczny tylko tam, gdzie koszula go nie zakrywa.
- Nad skórą: widoczny na gołym ciele.
- Ruch: chód, bieg, stanie, walka (różne akcje), wszystkie 8 kierunków (w kierunku 3 sprawdź Cloak).
- Jazda konna: akcje konne (23-29), tatuaż przesuwa się z ciałem.
- Paperdoll: własny i cudzy; po założeniu i zdjęciu paperdoll się odświeża; tatuaż pod Shirt.
- Zwłoki: po śmierci nie ma wyjątku w logu (`BAD_CONTAINER_LAYERS`), tatuaż (Blessed) zostaje, nie jest wyświetlany jako łup.
- Handel (`SecureTrade` 0x1F) w czasie noszenia tatuażu lewej ręki: brak artefaktów.
- Kliknięcie na postać: tatuaż nie jest klikalny (pixel-check kończy się na Mount; zamierzone).
