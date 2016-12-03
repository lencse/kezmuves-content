<!--
slug: tdd-esettanulmany-i-sudoku-megoldo
pubdate: 2016-11-21 12:00:00 +2
category: Kódolás
tags: [TDD, Typescript, Sudoku, Refactoring]
-->
![Rejtvényt fejtő úr](content/img/tdd-esettanulmany-i-sudoku-megoldo/sudoku.jpg) {.featured}

# TDD esettanulmány I. – Sudoku megoldó

Ismerkedem egy ideje a tesztvezérelt fejlesztéssel, de úgy érzem, van még mit tanulnom. Ebben a cikkben (és a következőkben) TDD módszereket követve próbálok kifejleszteni egy komponenst, ami backtracking algoritmussal old meg Sudoku rejtvényeket.

<!-- MORE -->

## Első lépések

A módszertan szempontjából szinte mindegy, de a választott programnyelv most a  Typescript. Ehhez szükségünk lesz [Node.js](https://nodejs.org/)-re, a tesztek futtatásához Mocha-t és Chai-t használunk.

````shell
npm init

npm install --save-dev typescript
npm install --save-dev typings
npm install --save-dev mocha
npm install --save-dev mocha-typescript
npm install --save-dev chai

node_modules/.bin/typings install --save chai
````

A kódokat a `/src`, a teszteket a `/test` mappában tároljuk, a lefordított js fileok pedig a `/dist` mappába kerülnek.

Szükségünk lesz egy `tsconfig.json`-re:

````json
{
    "compilerOptions": {
        "module": "commonjs",
        "target": "es2015",
        "removeComments": false,
        "moduleResolution": "node",
        "preserveConstEnums": true,
        "sourceMap": true,
        "experimentalDecorators": true,
        "declaration": true
    },
    "exclude": [
        "dist",
        "node_modules"
    ]
}
````

Egy pár hasznos script a `package.json`-ben:

````json
{
  "scripts": {
    "typings-install": "./node_modules/.bin/typings install",
    "build": "./node_modules/.bin/tsc -p . --outDir dist/",
    "test": "node_modules/.bin/mocha dist/test/",
    "build-test": "npm run build && npm test",
    "postinstall": "npm run typings-install"
  }
}
````

Érdemes kikényszeríteni az automatizált teszteket, ehhez nagy segítség egy `.travis.yml`:

````yml
language: node_js
node_js:
  - "4"
  - "5"
  - "6"
  - "7"
before_script:
  - npm run build
script:
  - npm test
````

Végre megírhatjuk első tesztünket. Ez egyelőre csak megpróbál példányosítani valamilyen Sudoku osztályt.

````typescript
// test/test.ts

import { suite, test, slow, timeout, skip, only } from "mocha-typescript";
import { assert } from "chai";
import { Sudoku } from "../src/Sudoku";

@suite class SudokuTest {

    @test "creation"() {
        let sudoku = new Sudoku();
        assert.isDefined(sudoku);
    }

}
````

````shell
npm run build-test
````

Természetesen bukik a teszt. Oldjuk meg!

````typescript
// src/Sudoku.ts

export class Sudoku {

}
````

Örülünk, Vincent?

````shell
  SudokuTest
    √ creation


  1 passing (0ms)
````
 Örülünk.

### Mi történt?

Jó sok mindent beszántottunk most a projektbe, érdemes tisztázni, hogy mi is történt.

* [npm](https://www.npmjs.com/): a Node.js csomagkezelője. Ez kezeli majd a külső függőségeinket, illetve segít a scriptek futtatásában.
* [Typescript](https://www.typescriptlang.org/): javascripten alapuló programnyelv, ami többek között típuskényszerítéssel, és OOP eszközökkel bővíti a js-t. Javascriptre fordul.
* [Typings](https://github.com/typings/typings#readme): a Typescript barátja. Természetesen szeretnénk használni a hatalmas mennyiségű, js-ben írt libraryt, de mivel a js-ben ninc szigorú típusosság, nem tudjuk, hogy milyen típusú válaszokta kapunk tőlük, az IDE nem tud kkódkiegészítést adni, stb. Ezen a problémán segít a Typings néhány megfelelő definíciós file elhelyezésével. Jelen projektben ilyen library a chai.
* [Mocha](https://mochajs.org/): Tesztfuttató keretrendszer javascripthez.
* [Chai](http://chaijs.com/): A Mocha hűséges párja, asserteket használhatunk vele
* [mocha-typescript](https://github.com/PanayotCankov/mocha-typescript#readme): Typescript alatt ezzel fottatjuk a scripteket, a példában már látható dekorátotokkal a tesztjeinkben.
* [Travis](https://travis-ci.org): Continous Integration eszköz. Ha projektünket a githubra töltjük fel, a Travis a megfelelő konfigurációs file segítségével automatikusan buildel és lefuttatja a tesztjeinket különböző környezeteken.

## A Sudoku megépítése

A TDD egyik előnye, hogy nem mindig kell nagyon alaposan megtervezni a szoftverünket. Elindulhatunk apró lépésekben, ha nem tudjuk pontosan, hogy kell végignézni a végtreméknek, akkor csak egy kis tesztet írunk egy ötlethez, ami most éppen jónak tűnik.

### Cellapozíció

Akárhogy is működik a Sudoku osztályunk, valamilyen módon szeretnénk majd hivatkozni a celláira. Mondjuk egy Position osztállyal. A sorokat és oszlopokat egytől indexeljük.

````typescript
    @test "position"() {
        let position = new Position(1, 2);
        assert.equal(1, position.getRow());
        assert.equal(2, position.getColumn());
    }
````

````typescript
export class Position {

    private row: number;
    private column: number;

    constructor (row: number, column: number) {
        this.row = row;
        this.column = column;
    }

    public getRow(): number {
        return this.row;
    }
    
    public getColumn(): number {
        return this.column;
    }
    
}
````

Ha importáljuk az új osztályt (ez innentől nem jelzem), a teszt ismét zöld.

Nagyszerű, de szeretnénk valami shortcutot a példányosításhoz.

````typescript
    @test "pos"() {
        let position = pos(1, 2);
        assert.equal(1, position.getRow());
        assert.equal(2, position.getColumn());
    }
````

````typescript
export function pos(row: number, column: number): Position {
    return new Position(row, column);
}
````
Így jóval kevesebbet kell gépelnünk. PROFIT.

### A sudoku mérete

Ideje, hogy beszéljünk a sudoku méretéről. Persze, ha meghalljuk, hogy sudoku, mindig a 9x9-es játékra gondolunk, de nem látom okát, hogy ilyen korlátok közé szorítsuk magunkat. A programunk nyugodtan lehet alkalmas 16x16-os, vagy 25x25-ös feladványok megoldásához is, arról nem is beszélve, hogy tesztelni, kipróbálni jóval egyszerűbb a 4x4-es változat. Igazából az 1x1-es sudokunak is van értelme, bár túl sok algoritmuselméleti kihívást nem tartogat.

Jelölje a bűvös N szám, hogy hány szám van egy résztáblázat egy sorában (a 9x9-es sudokunál N=3), és adjuk át ezt a konstruktorban!

````typescript
    @test "creation"() {
        let sudoku = new Sudoku(1);
        assert.isDefined(sudoku);
    }
````

````typescript
export class Sudoku {

    private n: number;

    constructor (n: number) {
        this.n = n
    }

}
````

### Értékek

Szeretnénk megjelölni egy cellát, és megnézni az értékét, egyelőre csak egy 1x1-es feladványban.

````typescript
    @test "put-and-val n=1"() {
        let sudoku = new Sudoku(1);
        sudoku.put(pos(1, 1), 1);
        assert.equal(1, sudoku.val(pos(1, 1)));
    }
````

Nagyon gyorsan akarom teljesíteni a tesztet, mókolok.

````typescript
export class Sudoku {

    private value: number;

    public put(position: Position, value: number) {
        this.value = value;
    }

    public val(position: Position): number {
        return this.value;
    }

}
````

Persze hamar kiderül a turpisság, ha csinálunk tesztesetet több cellára is.

````typescript
    @test "put-and-val n=2"() {
        let sudoku = new Sudoku(2);
        sudoku.put(pos(1, 1), 1);
        sudoku.put(pos(2, 1), 2);
        assert.equal(1, sudoku.val(pos(1, 1)));
        assert.equal(2, sudoku.val(pos(2, 1)));
    }
````
Most már meg kell csinálni rendesen, `value` helyett egy `cells` tömbbel. 
````typescript
export class Sudoku {

    private n: number;
    private cells: Array<number> = [];

    constructor (n: number) {
        this.n = n;
        for (let i = 0; i < Math.pow(n, 4); ++i) {
            this.cells.push(0);
        }
    }

    public put(position: Position, value: number) {
        this.cells[(position.getRow()-1) * Math.pow(this.n, 2) + position.getColumn()-1] = value;
    }

    public val(position: Position): number {
        return this.cells[(position.getRow()-1) * Math.pow(this.n, 2) + position.getColumn()-1];
    }

}
````
"Zöld csík" mellett megszüntetjük a kódduplikációt.
````typescript
export class Sudoku {

    public put(position: Position, value: number) {
        this.cells[this.transformPositionToIndex(position)] = value;
    }

    public val(position: Position): number {
        return this.cells[this.transformPositionToIndex(position)];
    }

    private transformPositionToIndex(position: Position): number {
        return (position.getRow()-1) * Math.pow(this.n, 2) + position.getColumn()-1;
    }

}
````

### A sudoku validálása


Tudnunk kell ellenőrizni, hogy szabályos-e a sudoku (nincs ütközés sorban, oszlopban, vagy résztáblázatban). Egy dolog biztos: a töküres sudoku még szabályos.
````typescript
    @test "valid-after-creation"() {
        let sudoku = new Sudoku(1);
        assert.isTrue(sudoku.isValid());
    }
````
Könnyű sikert aratunk.
````typescript
export class Sudoku {

    public isValid(): boolean {
        return true;
    }

}
````
A validálást kezdjük a sorokkal, nézzük meg, hogy jól reagál-e, ha egy sorban ütköző elemek vannak!
````typescript
    @test "check-invalid-row"() {
        let sudoku = new Sudoku(2);
        sudoku.put(pos(1, 1), 1);
        sudoku.put(pos(1, 3), 1);
        assert.isFalse(sudoku.isValid());
    }
````
Gyorsan beütünk egy triviális megoldást.
````typescript
export class Sudoku {

    public isValid(): boolean {
        for (let row = 1; row <= Math.pow(this.n, 2); ++row) {
            let vals = [];
            for (let col = 1; col <= Math.pow(this.n, 2); ++col) {
                const val = this.val(pos(row, col)) 
                if (val == 0) {
                    continue;
                }
                if (vals.indexOf(val) != -1) {
                    return false;
                }
                vals.push(val);
            }
        }
        return true;
    }

}
````
Ugyanígy vizsgáljuk az oszlopokat is.
````typescript
    @test "check-invalid-column"() {
        let sudoku = new Sudoku(2);
        sudoku.put(pos(1, 1), 1);
        sudoku.put(pos(3, 1), 1);
        assert.isFalse(sudoku.isValid());
    }
````

````typescript
export class Sudoku {

    public isValid(): boolean {
        for (let row = 1; row <= Math.pow(this.n, 2); ++row) {
           // ...
        }
        for (let col = 1; col <= Math.pow(this.n, 2); ++col) {
            let vals = [];
            for (let row = 1; row <= Math.pow(this.n, 2); ++row) {
                const val = this.val(pos(row, col)) 
                if (val == 0) {
                    continue;
                }
                if (vals.indexOf(val) != -1) {
                    return false;
                }
                vals.push(val);
            }
        }
        return true;
    }

}
````
A Clean Code könyvem ennél a pontnál rituális öngyilkosságot követett el, és nagy robajjal lezuhant a polcról. 

### Stratégia

Muszáj lesz megszüntetni a két ugyanolyan szerkezetű ciklust, már csak esztétikai okokból is, de főleg, mert ha ugyanezel az elvvel írnánk meg a résztáblázatok ellenőrzését, végképp káoszba fulladna az algoritmus.

Mit is csinálunk tulajdonképpen?

* Elindulunk az `(1, 1)` cellából.
* *Valamilyen szisztéma* szerint továbblépkedünk.
* Közben egy kosárba gyűjtögetjük a megtekintett számokat.
* *Néha* (pl. sor, vagy oszlop végén) ürítjük a kosarat
* Ha ütközést találunk, leállunk.
* Ha ütközés nélkül elértük a jobb alsó sarkot, leállunk, szabályos a sudoku.

Láthatjuk, hogy a bozonytalanságot jelölő *valamilyen* és *néha* szavakat leszámítva a lépések zöme megegyezik, függetlenül attól, hogy sorokat, oszlopokat, vagy résztáblázatokat vizsgálunk. Induljunk el ebbe az irányba.

A lépéseknél a validáció állapotának tárolásához létesítünk egy osztályt, ami tudja, hol járunk, és milyen számok vannak a kosárban.
````typescript
export class ValidationState {

    private position: Position;
    private seenValues: Array<number>;

    constructor (position: Position, seenValues: Array<number>) {
        this.position = position;
        this.seenValues = seenValues;
    }

    public getPosition(): Position {
        return this.position;
    }

    public getSeenValues(): Array<number> {
        return this.seenValues;
    }

}
````
Kel egy osztály, ami elvégzi a lépéseket. Ennek ismernie kell a `Sudoku` osztályt, hogy a léptetéshez tisztában legyen a sudoku méretével, illetve hogy épp milyen számot vizsgálunk. Ehhez először is láthatóvá kell tenni az N értéket.
````typescript
export class Sudoku {

    public getN(): number {
        return this.n;
    }

}
````
Először a sorok validálását végző léptetőt írjuk meg.
````typescript
export class RowValidatorIterator {

    private n: number;
    private sudoku: Sudoku;

    constructor (sudoku: Sudoku) {
        this.n = sudoku.getN();
        this.sudoku = sudoku;
    }

    public iterate(state: ValidationState): ValidationState {
    }

}
````
Ha a jobb alsó sarokban vagyunk, akkor sikeres a vizsgálat. Erre most nincs jobb ötletem, mint hogy a pozíciót a (nemlétező) `(0, 0)` helyre állítsuk.
````typescript
    public iterate(state: ValidationState): ValidationState {
        const p = state.getPosition();
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(0, 0), []);
        }
    }
````
Ha a sor végén vagyunk, ugrunk a következő elejére, és ürítjük a kosarat.
````typescript
    public iterate(state: ValidationState): ValidationState {
        const p = state.getPosition();
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(0, 0), []);
        }
        if (p.getColumn() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(p.getRow()+1, 1), []);
        }
    }
````
Különben pedig kosárba tesszük az épp vizsgált számot, és jobbra lépünk.
````typescript
    public iterate(state: ValidationState): ValidationState {
        const p = state.getPosition();
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(0, 0), []);
        }
        if (p.getColumn() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(p.getRow()+1, 1), []);
        }
        let newValues = state.getSeenValues();
        const val = this.sudoku.val(p);
        if (val != 0) {
            newValues.push(val);
        }
        return new ValidationState(pos(p.getRow(), p.getColumn()+1), newValues);
    }
````
Most már lecserélhetjük a beágyazott ciklust a sorok ellenőrzésénél.
````typescript
export class Sudoku {

    public isValid(): boolean {
        let state = new ValidationState(pos(1, 1), []);
        let iterator = new RowValidatorIterator(this);
        while (state.getPosition().getColumn() != 0) {
            if (state.getSeenValues().indexOf(this.val(state.getPosition())) != -1) {
                return false;
            }
            state = iterator.iterate(state);
        }
        for (let col = 1; col <= Math.pow(this.n, 2); ++col) {
            // ...
        }
        return true;
    }

}
````
Jók vagyunk, a tesztünk még mindig zöld.

Ahhoz, hogy továbblépjünk, generalizálni kell a sorellenőrző kódot.
````typescript
export abstract class ValidatorIterator {

    protected n: number;
    protected sudoku: Sudoku;

    constructor (sudoku: Sudoku) {
        this.n = sudoku.getN();
        this.sudoku = sudoku;
    }

    public abstract iterate(state: ValidationState): ValidationState;
    
}
````
````typescript
export class RowValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState): ValidationState {
        // ...
    }

}
````

````typescript
export class Sudoku {

    public isValid(): boolean {
        if (!this.validate(new RowValidatorIterator(this))) {
            return false;
        }
        for (let col = 1; col <= Math.pow(this.n, 2); ++col) {
           // ...
        }
        return true;
    }

    private validate(iterator: ValidatorIterator): boolean {
        let state = new ValidationState(pos(1, 1), []);
        while (state.getPosition().getColumn() != 0) {
            if (state.getSeenValues().indexOf(this.val(state.getPosition())) != -1) {
                return false;
            }
            state = iterator.iterate(state);
        }
        return true;
    }

}
````
Most már jöhet az oszlopvalidáló ciklus leváltása.
````typescript
export class ColumnValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState): ValidationState {
        const p = state.getPosition();
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(0, 0), []);
        }
        if (p.getRow() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(1, p.getColumn()+1), []);
        }
        let newValues = state.getSeenValues();
        const val = this.sudoku.val(p);
        if (val != 0) {
            newValues.push(val);
        }
        return new ValidationState(pos(p.getRow()+1, p.getColumn()), newValues);
    }

}
````

````typescript
export class Sudoku {

    public isValid(): boolean {
        return this.validate(new RowValidatorIterator(this))
            && this.validate(new ColumnValidatorIterator(this));
    }
    
}
````
Jöhet a résztáblázat ellenőrzése. Végre írunk új tesztet!
````typescript
    @test "check-invalid-subgrid"() {
        let sudoku = new Sudoku(2);
        sudoku.put(pos(3, 3), 1);
        sudoku.put(pos(4, 4), 1);
        assert.isFalse(sudoku.isValid());
    }
````

````typescript
export class SubgridValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState): ValidationState {
        const p = state.getPosition();
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new ValidationState(pos(0, 0), []);
        }
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() % this.n == 0) {
            return new ValidationState(pos(p.getRow()+1, 1), []);
        }
        if (p.getColumn() % this.n == 0 && p.getRow() % this.n == 0) {
            return new ValidationState(pos(p.getRow()-this.n+1, p.getColumn()+1), []);
        }
        let newValues = state.getSeenValues();
        const val = this.sudoku.val(p);
        if (val != 0) {
            newValues.push(val);
        }
        if (p.getColumn() % this.n == 0) {
            return new ValidationState(pos(p.getRow()+1, p.getColumn()-this.n+1), newValues);
        }
        return new ValidationState(pos(p.getRow(), p.getColumn()+1), newValues);
    }

}
````

````typescript
export class Sudoku {

    public isValid(): boolean {
        return this.validate(new RowValidatorIterator(this))
            && this.validate(new ColumnValidatorIterator(this))
            && this.validate(new SubgridValidatorIterator(this));
    }
    
}
````
### Elimináljunk!

Bár a Typescript egész szépen megoldotta a körkörös függőséget, nekem nem tetszik, hogy a `ValidatorIterator` tud a `Sudoku`-ról. Szüntessük meg!

Ehhez az iterálásnál át kell adnunk az épp vizsgált értéket, illetve az iterátornak továbbra is tudni kell, mekkora a sudoku.
````typescript
export abstract class ValidatorIterator {

    protected n: number;

    constructor (n: number) {
        this.n = n;
    }

    public abstract iterate(state: ValidationState, val: number): ValidationState;
    
}
````

````typescript
export class RowValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        // const val = this.sudoku.val(p); Ez a sor törlődik.
        // ...
    }

}

export class ColumnValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        // const val = this.sudoku.val(p); Ez a sor törlődik.
        // ...
    }

}

export class SubgridValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        // const val = this.sudoku.val(p); Ez a sor törlődik.
        // ...
    }

}
````
A `Sudoku`-ból törölhető a getN(), most már konstruktorban adjuk át. 
````typescript
export class Sudoku {

    // public getN(): number {
    //     return this.n;
    // }

    public isValid(): boolean {
        return this.validate(new RowValidatorIterator(this.n))
            && this.validate(new ColumnValidatorIterator(this.n))
            && this.validate(new SubgridValidatorIterator(this.n));
    }

    private validate(iterator: ValidatorIterator): boolean {
       // ...
        while (state.getPosition().getColumn() != 0) {
            // ...
            state = iterator.iterate(state, this.val(state.getPosition()));
        }
        // ...
    }

}
````

### Tisztítsunk!

Egy ponton nagyvonalúan hagytam egy kis csúnyaságot a rendszerben. Az, hogy a végállapotot a `(0, 0)` pozícióval jelöljük, átmenetileg jó volt, de semmiképp nem fenntartható megoldás. Ideje több állapotot jelölő osztályt bevezetni, a `ValidationState`-ből pedig interface lesz.

````typescript
export interface ValidationState {

    isFinished(): boolean;
    getPosition(): Position;
    getSeenValues(): Array<number>;

}
````
A `ValidationState` ozstályt átnevezzük. Egyelőre nem hozunk létre új osztályt, de meg kell valósítani az `isFinished()` metódust.
````typescript
export class InProgressValidationState implements ValidationState {

    public isFinished(): boolean {
        return this.position.getRow() != 0;
    }

}
````
````typescript
export class RowValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
       // ...
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new InProgressValidationState(pos(0, 0), []);
        }
        if (p.getColumn() == Math.pow(this.n, 2)) {
            return new InProgressValidationState(pos(p.getRow()+1, 1), []);
        }
        // ...
        return new InProgressValidationState(pos(p.getRow(), p.getColumn()+1), newValues);
    }

}
````
````typescript
export class ColumnValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new InProgressValidationState(pos(0, 0), []);
        }
        if (p.getRow() == Math.pow(this.n, 2)) {
            return new InProgressValidationState(pos(1, p.getColumn()+1), []);
        }
        // ...
        return new InProgressValidationState(pos(p.getRow()+1, p.getColumn()), newValues);
    }

}
````
````typescript
export class SubgridValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new InProgressValidationState(pos(0, 0), []);
        }
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() % this.n == 0) {
            return new InProgressValidationState(pos(p.getRow()+1, 1), []);
        }
        if (p.getColumn() % this.n == 0 && p.getRow() % this.n == 0) {
            return new InProgressValidationState(pos(p.getRow()-this.n+1, p.getColumn()+1), []);
        }
       // ...
        if (p.getColumn() % this.n == 0) {
            return new InProgressValidationState(pos(p.getRow()+1, p.getColumn()-this.n+1), newValues);
        }
        return new InProgressValidationState(pos(p.getRow(), p.getColumn()+1), newValues);
    }

}
````
Kell egy gyártófüggvény az induló állapotnak.
````typescript
export function startValidationState(): ValidationState {
    return new InProgressValidationState(pos(1, 1), []);
}
````
A `Sudoku`-ban kevés a változás. 
````typescript
export class Sudoku {

    private validate(iterator: ValidatorIterator): boolean {
        let state = startValidationState();
        while (!state.isFinished()) {
            // ...
        }
        // ...
    }

}
````
Most már létre tudjuk hozni az új osztályt a befejezett állapotnak.
````typescript
class FinishedValidationState implements ValidationState {

    public isFinished(): boolean {
        return true;
    }

    public getPosition(): Position {
        throw "Shouldn't be called";
    }

    public getSeenValues(): Array<number> {
        throw "Shouldn't be called";
    }

}
````

````typescript
class InProgressValidationState implements ValidationState {

    public isFinished(): boolean {
        return false;
    }

}
````

````typescript
export class RowValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new FinishedValidationState();
        }
        // ...
    }

}
````
````typescript
export class ColumnValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new FinishedValidationState();
        }
        // ...
    }

}
````
````typescript
export class SubgridValidatorIterator extends ValidatorIterator{

    public iterate(state: ValidationState, val: number): ValidationState {
        // ...
        if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
            return new FinishedValidationState();
        }
       // ...
    }

}
````
Mindhárom validáció ugyanúgy kezdődik: ellenőrizzük, hogy végeztünk-e. Érdemes ezt egy szinttel feljebb vinni az osztályhierarchiában.
````typescript
export abstract class ValidatorIterator {

    public iterate(state: ValidationState, val: number): ValidationState {
        if (state.getPosition().getColumn() == Math.pow(this.n, 2) && state.getPosition().getRow() == Math.pow(this.n, 2)) {
            return new FinishedValidationState();
        }
        return this.stepForward(state, val);
    }
    
    protected abstract stepForward(state: ValidationState, val: number): ValidationState;

}
````

````typescript
export class RowValidatorIterator extends ValidatorIterator{

    protected stepForward(state: ValidationState, val: number): ValidationState {
        // ...
        // if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
        //    return new FinishedValidationState();
        // }
        // ...
    }

}
````
````typescript
export class ColumnValidatorIterator extends ValidatorIterator{

    protected stepForward(state: ValidationState, val: number): ValidationState {
        // ...
        // if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
        //    return new FinishedValidationState();
        // }
        // ...
    }

}
````
````typescript
export class SubgridValidatorIterator extends ValidatorIterator{

    protected stepForward(state: ValidationState, val: number): ValidationState {
        // ...
        // if (p.getColumn() == Math.pow(this.n, 2) && p.getRow() == Math.pow(this.n, 2)) {
        //    return new FinishedValidationState();
        // }
        // ...
    }

}
````
És igazából a bukott validáció is lehet egy álllapot.
````typescript
export interface ValidationState {

    isFinished(): boolean;
    getPosition(): Position;
    getSeenValues(): Array<number>;
    isValid(): boolean;

}
````

````typescript

class InProgressValidationState implements ValidationState {

    public isValid(): boolean {
        return true;
    }

}
````

````typescript

class FinishedValidationState implements ValidationState {

    public isValid(): boolean {
        return true;
    }

}
````

````typescript
class InvalidValidationState implements ValidationState {

    public isFinished(): boolean {
        return false;
    }

    public getPosition(): Position {
        throw "Shouldn't be called";
    }

    public getSeenValues(): Array<number> {
        throw "Shouldn't be called";
    }

    public isValid(): boolean {
        return false;
    }

}
````

````typescript
export abstract class ValidatorIterator {

    public iterate(state: ValidationState, val: number): ValidationState {
        if (state.getSeenValues().indexOf(val) != -1) {
            return new InvalidValidationState();
        }
        // ...
    }
    
}
````

````typescript
export class Sudoku {

    private validate(iterator: ValidatorIterator): boolean {
        // ...
        while (!state.isFinished()) {
            if (!state.isValid()) {
                return false;
            }
            // ...
        }
        // ...
    }

}
````

### Álljunk megy egy pillanatra!

#### TDD ez még?

Jó kérdés, tényleg nem tudom. Nyilván feltűnt, hogy amikor a refaktorálási lépéseket végeztem, feltűnt, hogy nem írtam új teszteket, csak a meglévőket futtattam újra és újra, hogy ellenőrizzem, jó vagyok-e még. Az biztos, hogy lehetett volna inkább TDD szellemben csinálni ezt, az iterátorokhoz pl. tök jó teszteket lehetett volna írni előre. Úgy ítéltem meg, hogy ez a nagyságren még szűkösen épp belefér abba, hogy kis lépésekben haladjunk előre, ha megakadtam volna, akkor nekiálltam volna új teszteket írni.

Mindenestre a dolog filozófiájával kapcsolatban szívesen fogadok észrevételeket kommentben.

## Fújjunk egyet!

Azt hiszem, lehetne ezt tovább csinosítani, de úgy ítélem meg, hogy most pihenhetünk egyet. A validáció működik, a nagyon csúnya kódismétléseket kiszűrtük. Az állapot osztályokban kivételt dobunk, azt hiszem, ezt szebben kéne megoldani, de most nem látok triviális megoldást, hogyan.

Az alapozást elvégeztük, a következő cikkben jön a Sudoku feladvány megoldását megvalósító algoritmus.

### Forráskód

* A cikkben írt lépéseket [végig tudjátok követni a GitHubon.](https://github.com/lencse/afsudoku/commits/v0.1.0)
* Az elkészült forráskódot [itt lehet letölteni.](https://github.com/lencse/afsudoku/releases/tag/v0.1.0)
