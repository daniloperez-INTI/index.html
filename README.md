<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Escáner QR - Asistencia</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
  <script src="https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.js"></script>
</head>
<body class="bg-light">
<div class="container py-4 text-center" style="max-width:600px">
  <h2 class="mb-1">Control de Asistencia</h2>
  <p class="text-muted mb-4">Escáner QR</p>

  <button id="btnActivar" class="btn btn-primary btn-lg mb-3" onclick="activarCamara()">
    📷 Activar Cámara
  </button>
  <button id="btnDetener" class="btn btn-secondary btn-lg mb-3 d-none" onclick="detenerCamara()">
    ⏹ Detener
  </button>

  <div id="reader" style="width:100%;max-width:460px;margin:auto;"></div>

  <div class="mt-4">
    <label class="form-label fw-bold">Ingreso manual / Lector USB</label>
    <input type="text" id="nieInput" class="form-control w-75 mx-auto"
           placeholder="Escanea o escribe el NIE + Enter" autofocus>
  </div>

  <div id="resultado" class="mt-4 alert d-none"></div>
</div>

<script>
  // ⚠️ REEMPLAZA ESTA URL CON LA DE TU APPS SCRIPT
  var APPS_SCRIPT_URL = 'TU_URL_AQUI';

  var html5Qrcode = null;
  var procesando = false;

  function activarCamara() {
    document.getElementById('btnActivar').classList.add('d-none');
    document.getElementById('btnDetener').classList.remove('d-none');
    mostrarResultado('info', '⏳ Iniciando cámara...');

    html5Qrcode = new Html5Qrcode("reader");

    Html5Qrcode.getCameras()
      .then(function(cameras) {
        if (!cameras || cameras.length === 0) {
          mostrarResultado('danger', '❌ No se detectó ninguna cámara.');
          resetBotones();
          return Promise.reject('sin camara');
        }
        var camaraId = cameras[cameras.length - 1].id;
        return html5Qrcode.start(
          camaraId,
          { fps: 10, qrbox: { width: 250, height: 250 } },
          onScanSuccess,
          function() {}
        );
      })
      .then(function() {
        mostrarResultado('success', '✅ Cámara activa. Apunta al código QR.');
      })
      .catch(function(err) {
        if (err === 'sin camara') return;
        mostrarResultado('danger', '❌ Error de cámara: ' + err);
        resetBotones();
      });
  }

  function detenerCamara() {
    if (html5Qrcode) {
      html5Qrcode.stop().then(function() {
        html5Qrcode.clear();
        resetBotones();
        mostrarResultado('info', 'Cámara detenida.');
      });
    }
  }

  function resetBotones() {
    document.getElementById('btnActivar').classList.remove('d-none');
    document.getElementById('btnDetener').classList.add('d-none');
  }

  function onScanSuccess(decodedText) {
    if (procesando) return;
    procesando = true;
    registrarNIE(decodedText);
  }

  function registrarNIE(nie) {
    nie = nie.toString().trim();
    if (!nie) return;

    mostrarResultado('info', '⏳ Registrando: <strong>' + nie + '</strong>');

    var url = APPS_SCRIPT_URL + '?nie=' + encodeURIComponent(nie);

    fetch(url)
      .then(function(res) { return res.json(); })
      .then(function(data) {
        if (data.status === 'success') {
          mostrarResultado('success',
            '✅ <strong>' + data.nombre + '</strong><br>Sección: ' + data.seccion);
        } else {
          mostrarResultado('danger', '❌ ' + data.msg);
        }
        setTimeout(function() { procesando = false; }, 3000);
      })
      .catch(function(err) {
        mostrarResultado('danger', '❌ Error de conexión: ' + err);
        setTimeout(function() { procesando = false; }, 3000);
      });
  }

  function mostrarResultado(tipo, mensaje) {
    var div = document.getElementById('resultado');
    div.className = 'mt-4 alert alert-' + tipo;
    div.innerHTML = mensaje;
  }

  document.getElementById('nieInput').addEventListener('keypress', function(e) {
    if (e.key === 'Enter' && this.value.trim() && !procesando) {
      registrarNIE(this.value.trim());
      this.value = '';
    }
  });
</script>
</body>
</html>
