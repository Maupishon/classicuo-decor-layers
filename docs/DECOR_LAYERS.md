# Dekoracyjne layery (fork ClassicUO b13d3fe82, gałąź `feature/decor-layers`)

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

## E. ServUO: minimalna zmiana (nic nie zmieniono na serwerze; to opis)

Folder: `D:\UO\UO\NELDERIM\ServUO` (z `CLAUDE.md` serwera; ścieżka `D:\UO\ServUO-master\ServUO-master` z zadania nie istnieje).

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
5. Do sprawdzenia przed wdrożeniem (nie czytałem): skrypty z `Layer.Invalid + n` / pętlami po `Layer` (np. `Scripts/Nelderim/.../LevelItemManager.cs:121` rzutuje `(Layer)i`), `Possess.cs`, `Mannequin.cs`. `BulletinBoards.cs:378` używa `<= Layer.Mount`, więc nowych layerów nie obejmie.

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

## F. Przykład end-to-end: jeden testowy tatuaż

Użyj KOPII klienta (nie `C:\Nelderim\UO_70950\UO_70950`). Komendy pipeline z `CLAUDE.md` pipeline'u; ścieżki podmień.

1. Wolne ID itemu i animacji: `python nelderim.py search --client <KOPIA> --item "<nazwa>"`; wolne body HUMAN/EQUIPMENT jest też w `FREE_SLOTS.md` klienta.
2. Animacja: narysuj tatuaż jako ubranie (PNG RGBA) i zbuduj `.vd` 35 akcji x 5 kierunków dla ciała 400 jak zwykłe ubranie (`build_item.py` / `atlas_to_vd.py`, `vdtool info` = typ 2, 35x5). Wstrzyknięcie do anim5: `nelderim.py inject` / `wire` (dry-run, potem `--apply`).
3. Gump paperdolla: męski = AnimID + 50000, damski = + 10000 (`make_gump.py` / `nelderim.py gump3d`, zapis do `gumpartLegacyMUL.uop` przez `uop_gump_patch.py`).
4. Ikona art (`art.mul`/UOP wg klienta).
5. Tiledata: dla ItemID ustaw `layer = 0x20`, `animId = <body>`, flagę Wearable. NIEZWERYFIKOWANE: czy `nelderim_patch` potrafi ustawić pole layer. Jeśli nie, UOFiddler (Items, pole Layer). Klient wysyła przy zakładaniu layer z tiledata (`GameActions.cs:559`), więc tiledata musi mieć 0x20.
6. Equipconv.def (jeśli potrzebne): `400 <ItemID-animId> <conv> <gump> <hue>`.
7. ServUO: enum Layer + klasa `TattooRArm` z `ItemID` z kroku 1; restart serwera; `[add TattooRArm`, założyć.
8. Klient: skopiuj do kopii klienta `ClassicUO.exe`, `ClassicUO.Assets.dll`, `ClassicUO.IO.dll`, `ClassicUO.Renderer.dll`, `ClassicUO.Utility.dll` z `D:\UO\ClassicUO-build`.

### Co sprawdzić w grze
- Pod koszulą: załóż Shirt i tatuaż; tatuaż ma być widoczny tylko tam, gdzie koszula go nie zakrywa.
- Nad skórą: widoczny na gołym ciele.
- Ruch: chód, bieg, stanie, walka (różne akcje), wszystkie 8 kierunków (w kierunku 3 sprawdź Cloak).
- Jazda konna: akcje konne (23-29), tatuaż przesuwa się z ciałem.
- Paperdoll: własny i cudzy; po założeniu i zdjęciu paperdoll się odświeża; tatuaż pod Shirt.
- Zwłoki: po śmierci nie ma wyjątku w logu (`BAD_CONTAINER_LAYERS`), tatuaż (Blessed) zostaje, nie jest wyświetlany jako łup.
- Handel (`SecureTrade` 0x1F) w czasie noszenia tatuażu lewej ręki: brak artefaktów.
- Kliknięcie na postać: tatuaż nie jest klikalny (pixel-check kończy się na Mount; zamierzone).
