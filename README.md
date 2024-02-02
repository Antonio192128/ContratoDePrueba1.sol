// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract PrestamoDeFi {
    using SafeMath for uint256;

    struct Prestamo {
        uint id;
        uint monto;
        uint plazo;
        uint tiempoSolicitud;
        uint tiempoLimite;
        address prestatario; // Campo añadido
        bool aprobado;
        bool reembolsado;
        bool liquidado;
    }

    struct Cliente {
        bool activado;
        uint saldoGarantia;
        mapping(uint => Prestamo) prestamos;
        uint[] prestamoIds;
    }

    address public socioPrincipal;
    mapping(address => Cliente) public clientes;
    mapping(address => bool) public empleadosPrestamista;

    event SolicitudPrestamo(uint id, address indexed prestatario, uint monto, uint plazo);
    event PrestamoAprobado(uint id, address indexed prestatario, uint monto);
    event PrestamoReembolsado(uint id, address indexed prestatario, uint monto);
    event GarantiaLiquidada(uint id, address indexed prestatario);

    modifier soloSocioPrincipal() {
        require(msg.sender == socioPrincipal, "Solo el socio principal puede realizar esta accion");
        _;
    }

    modifier soloEmpleadoPrestamista() {
        require(empleadosPrestamista[msg.sender], "Solo un empleado prestamista puede realizar esta accion");
        _;
    }

    modifier soloClienteRegistrado() {
        require(clientes[msg.sender].activado, "Cliente no registrado");
        _;
    }

    constructor() {
        socioPrincipal = msg.sender;
        empleadosPrestamista[socioPrincipal] = true; // El socioPrincipal también es un prestamista
    }

    function altaPrestamista(address nuevoPrestamista) public soloSocioPrincipal {
        require(!empleadosPrestamista[nuevoPrestamista], "El prestamista ya esta dado de alta");
        empleadosPrestamista[nuevoPrestamista] = true;
    }

    function altaCliente(address nuevoCliente) public soloEmpleadoPrestamista {
        require(!clientes[nuevoCliente].activado, "El cliente ya esta dado de alta");
        clientes[nuevoCliente].activado = true;
        clientes[nuevoCliente].saldoGarantia = 0;
    }

    function depositarGarantia() public payable soloClienteRegistrado {
        require(msg.value > 0, "Debe enviar una cantidad positiva de Ether");
        clientes[msg.sender].saldoGarantia = clientes[msg.sender].saldoGarantia.add(msg.value);
    }

    function solicitarPrestamo(uint monto_, uint plazo_) public soloClienteRegistrado returns (uint) {
        Cliente storage cliente = clientes[msg.sender];
        require(cliente.saldoGarantia >= monto_, "Saldo de garantia insuficiente"); // Verifica que el cliente tenga suficiente garantia 

        uint nuevoId = cliente.prestamoIds.length + 1; //Calcula el nuevo ID basándose en la longitud del array de prestamos
        cliente.prestamoIds.push(nuevoId); //

        Prestamo storage nuevoPrestamo = cliente.prestamos[nuevoId]; //Crea un nuevo registro de Prestamo en el mapping de prestamos 
        nuevoPrestamo.id = nuevoId; // Establece los detalles del nuevo prestamo
        nuevoPrestamo.monto = monto_;
        nuevoPrestamo.plazo = plazo_;
        nuevoPrestamo.tiempoSolicitud = block.timestamp;
        nuevoPrestamo.prestatario = msg.sender; // Asignar el solicitante como el prestatario del nuevo préstamo 
        nuevoPrestamo.aprobado = false; //Inicialmente no aprobado
        nuevoPrestamo.reembolsado = false; // inicialmente no rembolsado
        nuevoPrestamo.liquidado = false; // Inicialmente no liquidado

        emit SolicitudPrestamo(nuevoId, msg.sender, monto_, plazo_); // Emite el evento correspondiente 

        return nuevoId; //Devuelve el ID del nuevo prestamo
    }

    function aprobarPrestamo(uint id_, address prestatario_) public soloEmpleadoPrestamista {
        Prestamo storage prestamo = clientes[prestatario_].prestamos[id_];
        require(prestamo.id != 0 && !prestamo.aprobado && prestamo.prestatario == prestatario_, "Prestamo no valido o ya aprobado");
        
        prestamo.aprobado = true;
        prestamo.tiempoLimite = block.timestamp.add(prestamo.plazo);

        emit PrestamoAprobado(id_, prestatario_, prestamo.monto);
    }

    function reembolsarPrestamo(uint id_) public soloClienteRegistrado {
        Cliente storage cliente = clientes[msg.sender];
        Prestamo storage prestamo = cliente.prestamos[id_];
        require(prestamo.id != 0 && prestamo.aprobado && !prestamo.reembolsado, "Prestamo no puede ser reembolsado");

        require(cliente.saldoGarantia >= prestamo.monto, "Saldo de garantia insuficiente");
        cliente.saldoGarantia = cliente.saldoGarantia.sub(prestamo.monto);
        prestamo.reembolsado = true;

        emit PrestamoReembolsado(id_, msg.sender, prestamo.monto);
        payable(socioPrincipal).transfer(prestamo.monto); // Realizar el transfer de ether al socioPrincipal
    }

    function obtenerPrestamosPorPrestatario(address prestatario_) public view returns (uint[] memory) {
        return clientes[prestatario_].prestamoIds;
    }
}
