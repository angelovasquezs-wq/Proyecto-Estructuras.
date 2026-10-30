Grafomón


pseudocodigo detras:

// ================= ESTRUCTURAS =================

ESTRUCTURA Grafomon
    nombre    : texto
    vidaMax   : entero
    vida      : entero
    ataque    : entero
FIN

ESTRUCTURA Ubicacion                 // nodo del grafo
    id          : entero
    nombre      : texto
    descripcion : texto
    oponente    : Grafomon           // NULO si ya no queda nadie o nunca hubo
    esJefe      : booleano           // true en la habitación final (oponente fuerte)
    vecino[4]   : arreglo de Ubicacion   // ARRIBA, ABAJO, IZQUIERDA, DERECHA
                                         // (NULO si no hay camino)
FIN

ESTRUCTURA Mapa
    nodos    : arreglo de Ubicacion
    cantidad : entero
    inicio   : Ubicacion
FIN

ESTRUCTURA Jugador
    equipo : Lista enlazada de Grafomon   // máximo 6
    activo : Grafomon                     // el que pelea en el combate
FIN

CONSTANTE MAX_EQUIPO ← 6


// ================= FUNCIONES AUXILIARES =================

FUNCION quedanVivos(equipo) -> booleano
    PARA CADA g EN equipo HACER
        SI g.vida > 0 ENTONCES RETORNAR VERDADERO
    RETORNAR FALSO
FIN

PROCEDIMIENTO curarEquipo(equipo)
    PARA CADA g EN equipo HACER
        g.vida ← g.vidaMax
FIN

PROCEDIMIENTO mostrarEquipo(equipo)
    i ← 1
    PARA CADA g EN equipo HACER
        mostrar i, g.nombre, "vida", g.vida, "/", g.vidaMax, "ataque", g.ataque
        i ← i + 1
FIN


// ================= CARGA DEL MAPA (CSV) =================

FUNCION crearGrafomon(texto) -> Grafomon
    // "Pikachu,35,8"  ->  nombre, vida, ataque
    partes ← separar(texto, ",")
    g.nombre  ← partes[0]
    g.vidaMax ← aEntero(partes[1])
    g.vida    ← g.vidaMax
    g.ataque  ← aEntero(partes[2])
    RETORNAR g
FIN

FUNCION crearUbicacion(fila) -> Ubicacion
    // fila[0]=ID, [1]=nombre, [2]=descripción, [3]=oponente (vacío si no hay),
    // [4..7]=IDs de Arriba, Abajo, Izquierda, Derecha, [8]=EsJefe ("Si"/"No")
    u.id          ← aEntero(fila[0])
    u.nombre      ← copiar(fila[1])
    u.descripcion ← copiar(fila[2])
    SI fila[3] está vacía ENTONCES u.oponente ← NULO
    SINO u.oponente ← crearGrafomon(fila[3])
    u.esJefe      ← (fila[8] == "Si")
    guardar los IDs de fila[4..7] para enlazarlos después
    RETORNAR u
FIN

FUNCION cargarMapa(archivo) -> Mapa
    SI no se puede abrir archivo ENTONCES
        mostrar "No se pudo abrir el archivo"
        RETORNAR NULO
    FIN

    mapa ← crearMapa()
    leerLinea(archivo)                         // saltar encabezado

    MIENTRAS hay más líneas HACER
        fila ← leerLinea(archivo)
        SI fila tiene menos de 9 campos ENTONCES CONTINUAR
        agregar crearUbicacion(fila) a mapa.nodos
    FIN

    // segunda pasada: crear las aristas
    PARA CADA nodo EN mapa.nodos HACER
        PARA dir DESDE 0 HASTA 3 HACER
            SI idVecino[dir] != -1 ENTONCES
                nodo.vecino[dir] ← buscarPorID(mapa, idVecino[dir])
        FIN
    FIN

    mapa.inicio ← mapa.nodos[0]
    RETORNAR mapa
FIN


// ================= COMBATE =================

// Retorna VERDADERO si el jugador gana, FALSO si pierde.
FUNCION combatir(jugador, enemigo) -> booleano
    mostrar "¡Aparece", enemigo.nombre, "!"

    MIENTRAS enemigo.vida > 0 Y quedanVivos(jugador.equipo) HACER

        // --- turno del jugador ---
        mostrar estado: jugador.activo y enemigo (vida actual)
        mostrar menú:
            1) Atacar
            2) Cambiar de grafomón
        opcion ← leer número

        SI opcion == 2 ENTONCES
            mostrarEquipo(jugador.equipo)
            n ← leer número
            nuevo ← n-ésimo grafomón del equipo
            SI nuevo es válido Y nuevo.vida > 0 ENTONCES
                jugador.activo ← nuevo
            SINO
                mostrar "No puedes elegir ese grafomón"
                CONTINUAR                      // no pierde el turno
        SINO
            enemigo.vida ← enemigo.vida - jugador.activo.ataque
            mostrar jugador.activo.nombre, "hace", jugador.activo.ataque, "de daño"
        FIN

        SI enemigo.vida <= 0 ENTONCES SALIR DEL CICLO

        // --- turno del enemigo ---
        jugador.activo.vida ← jugador.activo.vida - enemigo.ataque
        mostrar enemigo.nombre, "hace", enemigo.ataque, "de daño"

        SI jugador.activo.vida <= 0 ENTONCES
            mostrar jugador.activo.nombre, "quedó sin vida"
            SI quedanVivos(jugador.equipo) ENTONCES
                mostrar "Elige otro grafomón"
                jugador.activo ← pedir un grafomón del equipo con vida > 0
            FIN
        FIN
    FIN

    SI enemigo.vida <= 0 ENTONCES RETORNAR VERDADERO
    RETORNAR FALSO
FIN


// ================= ACCIONES DESPUÉS DE GANAR =================

PROCEDIMIENTO ofrecerCaptura(jugador, ubicacion)
    mostrar "¿Quieres capturar a", ubicacion.oponente.nombre, "? (1=Sí, 2=No)"
    SI respuesta != 1 ENTONCES RETORNAR

    SI tamaño(jugador.equipo) == MAX_EQUIPO ENTONCES
        mostrar "Tu equipo está lleno. Debes liberar uno para capturarlo."
        mostrarEquipo(jugador.equipo)
        n ← leer número (0 = cancelar)
        SI n == 0 ENTONCES RETORNAR
        g ← quitar el n-ésimo grafomón del equipo
        SI g == jugador.activo ENTONCES jugador.activo ← primer grafomón del equipo
        eliminar g
    FIN

    ubicacion.oponente.vida ← ubicacion.oponente.vidaMax
    agregarAlFinal(jugador.equipo, ubicacion.oponente)
    mostrar "¡Capturaste a", ubicacion.oponente.nombre, "!"
FIN


// ================= PANTALLA DE ESTADO =================

PROCEDIMIENTO mostrarEstado(ubicacion, jugador)
    mostrar ubicacion.nombre y ubicacion.descripcion
    SI ubicacion.oponente != NULO ENTONCES
        mostrar "Hay un grafomón salvaje:", ubicacion.oponente.nombre
    SINO
        mostrar "No hay nadie con quien pelear"
    mostrar "Tu equipo (", tamaño(jugador.equipo), "/", MAX_EQUIPO, "):"
    mostrarEquipo(jugador.equipo)
    mostrar direcciones disponibles (las que tengan vecino != NULO)
FIN


// ================= PARTIDA =================

FUNCION jugar(mapa) -> booleano          // VERDADERO si se quiere cerrar la app
    jugador.equipo ← crearLista()

    // elegir grafomón inicial
    mostrar "Elige tu grafomón inicial:"
    mostrar 3 iniciales (por ejemplo, definidos en el programa)
    agregarAlFinal(jugador.equipo, el elegido)
    jugador.activo ← el elegido

    actual ← mapa.inicio

    REPETIR
        mostrarEstado(actual, jugador)
        mostrar menú:
            1) Pelear
            2) Avanzar
            3) Cambiar grafomón activo
            4) Salir
        opcion ← leer número

        SEGÚN opcion HACER

            1: SI actual.oponente es NULO ENTONCES
                   mostrar "No hay nadie con quien pelear"
               SINO
                   gano ← combatir(jugador, actual.oponente)
                   SI NO gano ENTONCES
                       mostrar "Perdiste el combate. ¡FIN DEL JUEGO!"
                       RETORNAR FALSO
                   FIN

                   SI actual.esJefe ENTONCES
                       mostrar "¡Derrotaste al jefe! ¡GANASTE EL JUEGO!"
                       mostrarEquipo(jugador.equipo)
                       RETORNAR FALSO
                   FIN

                   curarEquipo(jugador.equipo)
                   ofrecerCaptura(jugador, actual)
                   actual.oponente ← NULO       // ya no queda oponente aquí
               FIN

            2: dir ← leer dirección (1 a 4) - 1
               SI actual.vecino[dir] es NULO ENTONCES
                   mostrar "Dirección no válida"
               SINO
                   actual ← actual.vecino[dir]
               FIN

            3: mostrarEquipo(jugador.equipo)
               n ← leer número
               SI n es válido Y n-ésimo grafomón tiene vida > 0 ENTONCES
                   jugador.activo ← n-ésimo grafomón

            4: RETORNAR VERDADERO

            OTRO: mostrar "Opción inválida"
        FIN
    HASTA QUE termine
FIN


// ================= PROGRAMA PRINCIPAL =================

PROCEDIMIENTO principal()
    archivo ← "grafomon.csv"
    mapa ← NULO

    REPETIR
        mostrar menú:
            1) Cargar mapa desde CSV
            2) Iniciar partida
            3) Salir
        opcion ← leer número

        SEGÚN opcion HACER
            1: mapa ← cargarMapa(archivo)
               SI mapa != NULO ENTONCES mostrar "Mapa cargado"
            2: SI mapa es NULO ENTONCES
                   mostrar "Primero carga el mapa"
               SINO
                   mapa ← cargarMapa(archivo)   // mapa fresco: los oponentes reaparecen
                   SI jugar(mapa) ENTONCES RETORNAR
               FIN
            3: RETORNAR
            OTRO: mostrar "Opción inválida"
        FIN
    HASTA QUE termine
FIN

