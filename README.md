# Socket Server Basico

Un servidor de Websockets usando Node, Express y Socket.io

Temas cubiertos en mi curso de Node de cero a experto

# Sockets - Aplicación de cola

- Clase Centralizar la lógica de los tickets
    - Se crea una clase con la lógica de los tickets, en los cuales el constructor va inicializar los valores para poder funcionar la aplicación de tickets.
    - Además creamos  las funciones correspondientes para hacer funcionar la tickeria como si es el mismo día al cual se esta conectado entonces se cargan los datos de la BD en caso contrario se guarda en BD

    ```jsx
    const path = require('path');
    const fs   = require('fs');

    class TicketControl {

        constructor() {

            this.ultimo   = 0;
            this.hoy      = new Date().getDate();
            this.tickets  = [];
            this.ultimos4 = [];

            this.init();
        }

        get toJson() {
            return {
                ultimo: this.ultimo,
                hoy: this.hoy,
                tickets: this.tickets,
                ultimos4: this.ultimos4
            }
        }

        init() {

            const {hoy, tickets, ultimo, ultimos4} = require('../db/data.json');
            if( hoy == this.hoy ){
                this.tickets  = tickets;
                this.ultimo   = ultimo;
                this.ultimos4 = ultimos4;
            } else {
                // Es otro día
                this.guardarDB();
            }
        }

        guardarDB() {
            const dbPath = path.join( __dirname, '../db/data.json' );
            fs.writeFileSync( dbPath, JSON.stringify( this.toJson ) );
        }

    }

    module.exports = TicketControl;
    ```

    - Cuando la clase inicia se cargan los datos que están en el `json` , y si es el mismo día actualiza sus atributos, en caso contrario lo guarda en la bd.
- Modelo - Siguiente y atender nuevo ticket
    - Para trabajar con los tickets se va crear otras funciones en la clase `atenderTicket` `siguiente` y vamos a crear la clase `Ticket` con el fin de terminar las últimas funcionalidades de la ticketera.
    - La función `siguiente` va a sumar en uno el atributo de la clase `ultimo` y creará un ticket la cual pushearemos a los tickets y se guarda en DB, esta función retornará el numero de ticket que esté

    ```jsx
    siguiente() {
            this.ultimo += 1;
            const ticket = new Ticket( this.ultimo, null );
            this.tickets.push( ticket );

            this.guardarDB();

            return 'Ticket ' + ticket.numero;
        }
    ```

    - En atender el siguiente Ticket, primero se va comprobar si en los tickets está lleno, luego se eliminar el que se atendió y si pone al final el nuevo ticket en la fila de tickets. Después tenemos un array de los 4 últimos en donde se agregar el ticket.

    ```jsx
    atenderTicket( escritorio ){

            // No tenemos tickets
            if( this.tickets.length === 0 ) {
                return null;
            }

            const ticket = this.tickets.shift();
            ticket.escritorio = escritorio;

            this.ultimos4.unshift( ticket );

            if( this.ultimos4.length > 4 ){
                this.ultimos4.splice(-1,1);
            }

            this.guardarDB();

            return ticket;
        }
    ```

    - En el caso de la clase de ticket que se hablo es muy simple, solo tiene 2 atributos: el de numero y en que escritorio va ser ocupado.

    ```jsx
    class Ticket {
        constructor( numero, escritorio) {
            this.numero     = numero;
            this.escritorio = escritorio;
        }
    }
    ```

- Socket: Siguiente Ticket
    - Se va trabajar en la página los tickets, lo que se va hacer es ir al servidor y alistar los eventos, tanto cuando el cliente este ingresando a ver la cola de tickets como cuando le de siguiente al ticket. En el caso del back-end se va emitir dos eventos `socket.emit` y `socket.on` , en el primer caso se creará un evento con el nombre de último ticket y se le enviará el ultimo ticket del controlador de tickets y para el siguiente evento, que controla la cola de tickets se va a usar la función de siguiente en el control de tickets, y por último ejecutará el `callback` que recibe del cliente.

    ```jsx
    const socketController = (socket) => {
        
        socket.emit( 'ultimo-ticket', ticketControl.ultimo );

        socket.on('siguiente-ticket', ( payload, callback ) => {
        
            const siguiente = ticketControl.siguiente();
            callback(siguiente);

            // Todo: Notificar que hay un nuevo ticket pediente de asignar
        })
    }
    ```

    - En el lado del ciente se enciende el evento del último ticket y se rellena el input de la pantalla con el último ticket. Cuando se inicie el evento del botón crear va emitir un evento del socket llamado siguiente ticket lo cual va comunicarse con el back y actualiza el input.

    ```jsx
    socket.on('ultimo-ticket', (ultimo) => {

        lblNuevoTicket.innerText = 'Ticket ' + ultimo;
    })

    btnCrear.addEventListener( 'click', () => {
        
        socket.emit( 'siguiente-ticket', null, ( ticket ) => {
            lblNuevoTicket.innerText = ticket;
        });
    });
    ```

- Preparar pantalla de escritorio
    - En el js de escritorio, es el que se va encargar de administrar los tickets por cada escritorio, primero a través del URL se va buscar el nombre del escritorio y si no tiene eso se va enviarlo al index. Una vez que se ha hecho la verificación se va trabajar con los demás eventos

    ```jsx
    //Referencias Html
    const lblEscritorio = document.querySelector('h1');
    const btnAtender    = document.querySelector('button');

    console.log('Escritorio HTML');

    const searchParams = new URLSearchParams( window.location.search );

    if( !searchParams.has('escritorio') ) {
        window.location = 'index.html';
        throw new Error('El escritorio es obligatorio');
    }

    const escritorio = searchParams.get('escritorio');
    lblEscritorio.innerText = escritorio;

    const socket = io();

    socket.on('connect', () => {
        // console.log('Conectado');
        btnAtender.disabled = false;
    });

    socket.on('disconnect', () => {
        // console.log('Desconectado del servidor');
        btnAtender.disabled = true;
    });

    socket.on('ultimo-ticket', (ultimo) => {
        
        // lblNuevoTicket.innerText = 'Ticket ' + ultimo;
    })

    btnAtender.addEventListener( 'click', () => {
        
        // socket.emit( 'siguiente-ticket', null, ( ticket ) => {
        //     lblNuevoTicket.innerText = ticket;
        // });
    });
    ```

- Socket: Atender Ticket
    - Para atender el siguiente ticket se va trabajar con el botón `btnAtender` al momento de darle click este va conectarse con el evento `atender-ticket` y se le va enviar el numero del escritorio que va querer atenderlo, luego le enviamos la función de respuesta, la cual va verificar que todo este bien y pintar los datos en el front dependiendo de la respuesta.

    ```jsx
    btnAtender.addEventListener( 'click', () => {
        
        socket.emit( 'atender-ticket', { escritorio } ,( { ok, ticket, msg } ) => {
            
            if( !ok ) {
                lblTicket.innerText = 'Nadie.'
                return divAlerta.style.display = '';
            } 

            lblTicket.innerText = `Ticket ${ticket.numero}` ;

        })

        // socket.emit( 'siguiente-ticket', null, ( ticket ) => {
        //     lblNuevoTicket.innerText = ticket;
        // });
    });
    ```

    - En el caso del `back-end` estamos recibiendo el evento lo cual se van a usar las funciones y validaciones para retornar los datos.

    ```jsx
    socket.on( 'atender-ticket', ({ escritorio }, callback) => {
            
            if( !escritorio ) {
                return callback({
                    ok: false,
                    msg: 'Es escritorio es obligatorio'
                });
            }

            const ticket = ticketControl.atenderTicket( escritorio );
            if( !ticket ) {
                callback( { ok: false, msg: 'Ya no hay tickets pendientes'});
            } else {
                callback( { ok: true, ticket});
            }
        });
    ```

- Mostrar cola de tickets en pantalla
    - Para mostrar la cola de tickets en la pantalla vamos a utilizar los 4 últimos tickets del total, lo cual tenemos que configurar en el `back-end` el cual va emitir la información y `front-end` que va a recibir la información cuando se active un evento como el de atender ticket.

    ```jsx
    const socketController = (socket) => {
        
        socket.emit( 'ultimo-ticket', ticketControl.ultimo );
        socket.emit( 'estado-actual', ticketControl.ultimos4 );
    ```

    ```jsx
    //Referencias

    const lblTicket1 = document.querySelector('#lblTicket1')
    const lblEscritorio1 = document.querySelector('#lblEscritorio1')

    const lblTicket2 = document.querySelector('#lblTicket2')
    const lblEscritorio2 = document.querySelector('#lblEscritorio2')

    const lblTicket3 = document.querySelector('#lblTicket3')
    const lblEscritorio3 = document.querySelector('#lblEscritorio3')

    const lblTicket4 = document.querySelector('#lblTicket4')
    const lblEscritorio4 = document.querySelector('#lblEscritorio4')

    console.log('Público HTML')

    const socket = io();

    socket.on('estado-actual', ( payload ) => {
        
        const [ ticket1, ticket2, ticket3, ticket4 ] = payload;

      

        if( ticket1 ){
            lblTicket1.innerText = 'Ticket '+ ticket1.numero;
            lblEscritorio1.innerText = ticket1.escritorio;
        }

        if( ticket2 ){
            lblTicket2.innerText = 'Ticket '+ ticket2.numero;
            lblEscritorio2.innerText = ticket2.escritorio;
        }

        if( ticket3 ){
            lblTicket3.innerText = 'Ticket '+ ticket3.numero;
            lblEscritorio3.innerText = ticket3.escritorio;
        }

        if( ticket4 ){
            lblTicket4.innerText = 'Ticket '+ ticket4.numero;
            lblEscritorio4.innerText = ticket4.escritorio;
        }

    });
    ```

- Tickets pendientes por atender
    - En este caso vamos a mostrar en los escritorios los tickets pendientes que quedan atender, este evento se va actualizar cuando se genere un nuevo ticket, atienda un ticket y cuando se inicia la ventana del escritorio. Sencillamente el cliente va recepcionar la información y el `back-end` va emitir la data cuando los eventos aparezcan.

    ```jsx
    socket.on('tickets-pendientes', (tickets) => {

        lblPendientes.innerText = tickets;    
    })
    ```

    ```jsx
    const socketController = (socket) => {
        
        socket.emit( 'ultimo-ticket', ticketControl.ultimo );
        socket.emit( 'estado-actual', ticketControl.ultimos4 );
        socket.emit( 'tickets-pendientes', ticketControl.tickets.length);

    socket.on('siguiente-ticket', ( payload, callback ) => {
        
            const siguiente = ticketControl.siguiente();
            callback(siguiente);
            socket.broadcast.emit( 'tickets-pendientes', ticketControl.tickets.length);

            // Todo: Notificar que hay un nuevo ticket pediente de asignar
        });
    ...
    socket.on( 'atender-ticket', ({ escritorio }, callback) => {
    ....
    	socket.emit( 'tickets-pendientes', ticketControl.tickets.length);
    	socket.broadcast.emit( 'tickets-pendientes', ticketControl.tickets.length);

    .....
    ```

- Reproducir audio cuando se asigna un ticket
    - Asignamos un sonido cuando se atiende un nuevo ticket, esto no se puede hacer en chrome por politicas el cual no se puede activar un evento del doom a menos que el usuario lo este haciendo. Mozilla si lo permite.

    ```jsx
    socket.on('estado-actual', ( payload ) => {
        
        const audio = new Audio('./audio/new-ticket.mp3');
        audio.play();
    ```