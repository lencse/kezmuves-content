# TDD esettanulmány I. – A Sudoku megfejtése

Az [előző részben](https://kezmuvesprogramozo.hu/2016/11/21/tdd-esettanulmany-i-sudoku-megoldo) ott hagytuk abba, hogy van egy `Sudoku` osztályunk, ami tudja magát ellenőrizni, hogy épp szabályos állapotban van-e. Most már csak az algoritmust kell megírnunk, ami egy hiányosan kitöltött Sudokut megold. Természetesen most is szigorúan TDD elvek szerint programozunk.

## A megoldóalgoritmus

Mielőtt kódolni kezdünk, beszéljünk kicsit a megoldóalgoritmusról. A következő egy elég egyszerű, visszalépéses megoldáskereső:

!!!KÉP!!!

Ennyi, ki lehet próbálni papírral-ceruzával. Most már csak az a kérdés, hogyan kódoljuk le.

## A megoldás

Egyelőre nem tudjuk, hogy fogjuk lekódolni a megoldást. Egy biztos: *valahogy el kell kezdeni*. Legyen egy `startSolving` metódus, ami egy megoldás alatt lévő sudokuval tér vissza! Ehhez egy teszt:

````typescript
    @test "solving"() {
        let sudoku = new Sudoku(2);
        sudoku.put(pos(1, 2), 2);
        sudoku.put(pos(2, 1), 3);
        let solving = sudoku.startSolving();
        assert.isDefined(solving);
    }
````
Egyelőre nem bonyolítjuk túl:
````typescript
export class Sudoku {

    public startSolving(): Sudoku {
        return this;
    }

}
````
Nincs mese, valahogy meg kell különböztetnünk, hogy a sudoku táblát most éppen megoldjuk (`SudokuUnderSolving`), vagy még csak beállítjuk rajta a feladatot (`SudokuUnderSetup`). Ehhez absztrakttá tesszük a `Sudoku` osztályt. De hogy ezt megtehessük, sehol nem szabad közvetlenül példányosítani, inkább használjunk gyártófüggvényt! Sajnos a tesztekben elég sok cserét kell végrehajtani.

````typescript
@suite class SudokuTest {

    @test "creation"() {    
        let sudoku = Sudoku.create(1);
        // ...
    }

    @test "solving"() {
        let sudoku = Sudoku.create(2);
        // ...
    }

    @test "put-and-val n=1"() {
        let sudoku = Sudoku.create(1);
        // ...
    }

    @test "put-and-val n=2"() {
        let sudoku = Sudoku.create(2);
        // ...
    }

    @test "valid-after-creation"() {
        let sudoku = Sudoku.create(1);
        // ...
    }

    @test "check-invalid-row"() {
        let sudoku = Sudoku.create(2);
        // ...
    }

    @test "check-invalid-column"() {
        let sudoku = Sudoku.create(2);
        // ...
    }
    
    @test "check-invalid-subgrid"() {
        let sudoku = Sudoku.create(2);
        // ...
    }

    @test "position"() {
        let position = new Position(1, 2);
        assert.equal(1, position.getRow());
        assert.equal(2, position.getColumn());
    }

    @test "pos"() {
        let position = pos(1, 2);
        assert.equal(1, position.getRow());
        assert.equal(2, position.getColumn());
    }

}
````
A `Sudoku` osztályban bevezetjük a gyártófüggvényt, és priváttá tesszük a konstruktort.
````typescript
export class Sudoku {

    public static create(n: number): Sudoku {
        return new Sudoku(n);
    }
    
    private constructor (n: number) {
        // ...
    }
}
````

Most már absztrakt lehet az osztályunk. Amit eddig `Sudoku`ként ismertünk, az valójában a `SudokuUnderSetup`

````typescript
export abstract class Sudoku {

    public static create(n: number): Sudoku {
        return new SudokuUnderSetup(n);
    }

    protected constructor (n: number) {
        // ...
    }

}

class SudokuUnderSetup extends Sudoku {

}
````

A `put` metódusnak igazából ebben az alosztályban a helye.

````typescript
export abstract class Sudoku {

    protected n: number;
    protected cells: Array<number> = [];

    protected transformPositionToIndex(position: Position): number {
        // ...
    }
}

export class SudokuUnderSetup extends Sudoku {

    public put(position: Position, value: number) {
        // ...
    }

}
````

Bevezetjük a `SudokuUnderSolving` osztályt, ez kezdetben egy másolat lesz az eredeti tábláról. Módosítjuk a tesztet is, hogy ezt ellenőrizzük.

````typescript
    @test "solving"() {
        // ...
        assert.equal(2, solving.val(pos(1, 2)));
    }
````
Az új alosztály megkapja a konstruktorban a számokat.
````typescript
export abstract class Sudoku {

    protected constructor (n: number, cells: Array<number> = null) {
        this.n = n;
        for (let i = 0; i < Math.pow(n, 4); ++i) {
            this.cells.push(cells ? cells[i] : 0);
        }
    }

    public startSolving(): SudokuUnderSolving {
        return new SudokuUnderSolving(this.n, this.cells);
    }

}

export class SudokuUnderSolving extends Sudoku {

}
````
Sokat tökvakarásztunk most az osztályhierarchiában, ideje elkezdeni a megoldást. Az algoritmus szerint az első lépés a tesztsudoku esetében az, hogy a bal felső sarokba berakunk egy egyest.
````typescript
    @test "solving"() {
        let sudoku = Sudoku.create(2);
        sudoku.put(pos(1, 2), 2);
        sudoku.put(pos(2, 1), 3);
        let solving = sudoku.startSolving();
        assert.equal(2, solving.val(pos(1, 2)));
        let step1 = solving.step();
        assert.equal(1, step1.val(pos(1, 1)));
    }
````
Gyors zöld csíkra törekszünk, meg se próbáljuk rendesen megoldani.
````typescript
export class SudokuUnderSolving extends Sudoku {

    public step(): SudokuUnderSolving {
        let next = new SudokuUnderSolving(this.n, this.cells);
        next.cells[0] = 1;
        return next;
    }

}
````
Ebben a rövid sorban (`next.cells[0] = 1;`) túl sok hardcode van, nézzük meg, mik is ezek!
* **Miért 0?** – Azért, mert ez az első szabadon variálható cella indexe.
* **Miért 1?** – Azért, mert a cellában eddig 0 állt, ezt növeljük eggyel.

Tárolnunk kell tehát:
*  A szabad cellákat
*  És hogy ezek közül épp melyiken állunk

````typescript
export class SudokuUnderSolving extends Sudoku {

    private modifyableCells: Array<number>;
    private current: number;

}
````
Jó lenne ezeknek a `SudokuUnderSolving` létrehozásánál értéket adni. Ehhez egy absztrakt `setUp` metódust hívunk a szülőosztály konstruktorában.
````typescript
export abstract class Sudoku {

    protected constructor (n: number, cells: Array<number> = null) {
        // ...
        this.setUp();
    }

    protected abstract setUp();

}
````
Az egyik alosztályban ez lehet üres.
````typescript
export class SudokuUnderSetup extends Sudoku {

   protected setUp() {
    }

}
````
A `SudokuUnderSolving` esetében pedig beállítjuk a kezdőértékeket, és kicseréljük a beégetett számokat.
````typescript
export class SudokuUnderSolving extends Sudoku {

    protected setUp() {
        this.modifyableCells = [];
        this.current = 0;
        this.cells.map((cell: number, idx: number) => {
            if (cell == 0) {
                this.modifyableCells.push(idx);
            }
        });
    }

    public step(): SudokuUnderSolving {
        let next = new SudokuUnderSolving(this.n, this.cells);
        next.modifyableCells = this.modifyableCells;
        next.current = this.current;
        next.cells[next.modifyableCells[next.current]] = 1;
        return next;
    }

}
````
Vegyük észre, hogy a `current` a `modifyableCells`-en belüli indexre mutat, és az mutat a cella valódi indexére.

A fenti kódban az 1 még mindig hardcode.
````typescript
export class SudokuUnderSolving extends Sudoku {

    public step(): SudokuUnderSolving {
        // ...
        next.cells[next.modifyableCells[next.current]] = 1;
        return next;
    }

}
````
````typescript
````
````typescript
````
````typescript
````
````typescript
````


