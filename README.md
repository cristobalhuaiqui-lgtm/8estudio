<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>8E | Gestión Oficial</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src='https://cdn.jsdelivr.net/npm/fullcalendar@6.1.8/index.global.min.js'></script>
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .fc-daygrid-day { cursor: pointer !important; position: relative; }
        .hay-prueba { background-color: #fff1f2 !important; }
        .hay-prueba::after {
            content: '●';
            position: absolute;
            bottom: 2px;
            left: 50%;
            transform: translateX(-50%);
            color: #ef4444;
            font-size: 12px;
        }
        .admin-only { display: none; }
        .es-admin .admin-only { display: flex; }
    </style>
</head>
<body class="pb-20">

    <nav class="p-4 bg-white border-b flex justify-between items-center max-w-6xl mx-auto rounded-b-2xl shadow-sm mb-4">
        <h1 class="text-lg font-black uppercase tracking-tighter">Estudio<span class="text-blue-600">8E</span></h1>
        <button onclick="activarAdmin()" class="text-[9px] font-bold bg-slate-100 px-4 py-2 rounded-full uppercase tracking-widest">🔒 Acceso Directiva</button>
    </nav>

    <main class="max-w-6xl mx-auto px-4 grid grid-cols-1 lg:grid-cols-12 gap-6">
        
        <div class="lg:col-span-7">
            <div class="bg-white p-2 rounded-3xl shadow-sm border border-slate-100 mb-6">
                <div id="calendar"></div>
            </div>
            <section>
                <h2 class="text-[10px] font-bold uppercase text-slate-400 mb-3 tracking-widest px-2 italic">Próximas Evaluaciones</h2>
                <div id="lista-pruebas" class="space-y-2"></div>
            </section>
        </div>

        <div class="lg:col-span-5 space-y-6">
            <div id="panel-admin" class="admin-only flex-col bg-slate-900 p-6 rounded-[32px] text-white shadow-xl">
                <p id="txt-fecha" class="text-blue-400 text-[10px] font-bold mb-4 uppercase text-center italic">Selecciona un día en el calendario</p>
                
                <div class="space-y-4">
                    <div class="bg-white/5 p-3 rounded-2xl border border-white/10">
                        <p class="text-[8px] uppercase font-bold text-slate-500 mb-2 italic">Añadir Prueba</p>
                        <input type="text" id="add-prueba" placeholder="Ej: Control de Matemáticas" class="w-full bg-transparent border-b border-white/20 p-2 text-sm outline-none mb-2 focus:border-blue-400">
                    </div>

                    <div class="bg-white/5 p-3 rounded-2xl border border-white/10">
                        <p class="text-[8px] uppercase font-bold text-slate-500 mb-2 italic">Subir Apuntes</p>
                        <input type="text" id="add-materia" placeholder="Materia..." class="w-full bg-transparent border-b border-white/20 p-2 text-sm outline-none mb-3 focus:border-blue-400">
                        <label class="block w-full bg-white text-black py-3 rounded-xl text-center font-bold text-[10px] uppercase cursor-pointer shadow-lg active:scale-95 transition-all">
                            📸 Tomar Foto
                            <input type="file" accept="image/*" capture="environment" class="hidden" onchange="procesarFoto(event)">
                        </label>
                    </div>
                    
                    <button onclick="sincronizarTodo()" id="btn-sync" class="w-full bg-blue-600 py-4 rounded-2xl font-bold text-xs uppercase shadow-lg hover:bg-blue-700 transition-all">Sincronizar y Publicar</button>
                </div>
            </div>

            <section>
                <h2 class="text-[10px] font-bold uppercase text-slate-400 mb-3 tracking-widest px-2 italic">Apuntes del Día</h2>
                <div id="lista-fotos" class="space-y-4"></div>
            </section>
        </div>
    </main>

    <script>
        const BIN_ID = "69bdbb37c3097a1dd54428b9"; // TU NUEVO ID
        const MASTER_KEY = "$2a$10$.aFj61oAM40O0GhTX.LR0.TazociioZhdYdgu/O9gWx8lMdpObBQm";

        let db = { pruebas: [], apuntes: {} };
        let diaSel = null;
        let fotosSesion = [];

        function activarAdmin() {
            if(prompt("Clave Directiva:") === "8e2025directiva") {
                document.body.classList.add('es-admin');
                descargarDatos();
            }
        }

        function procesarFoto(e) {
            const mat = document.getElementById('add-materia').value || "Materia";
            const reader = new FileReader();
            reader.onload = function(f) {
                const img = new Image();
                img.onload = function() {
                    const canvas = document.createElement('canvas');
                    const ctx = canvas.getContext('2d');
                    const scale = 800 / img.width;
                    canvas.width = 800; canvas.height = img.height * scale;
                    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
                    fotosSesion.push({ m: mat, i: canvas.toDataURL('image/jpeg', 0.6) });
                    alert("Foto lista. No olvides Sincronizar.");
                }
                img.src = f.target.result;
            }
            reader.readAsDataURL(e.target.files[0]);
        }

        async function sincronizarTodo() {
            const btn = document.getElementById('btn-sync');
            btn.innerText = "Sincronizando...";
            btn.disabled = true;

            try {
                // 1. Bajamos lo último para no borrar nada por accidente
                const resGet = await fetch(`https://api.jsonbin.io/v3/b/${BIN_ID}/latest`, {
                    headers: { 'X-Master-Key': MASTER_KEY, 'X-Bin-Meta': false }
                });
                db = await resGet.json();

                // 2. Si escribiste una prueba, la añadimos
                const nombreP = document.getElementById('add-prueba').value;
                if(diaSel && nombreP) {
                    db.pruebas.push({ id: Date.now(), n: nombreP, f: diaSel });
                    document.getElementById('add-prueba').value = "";
                }

                // 3. Si tomaste fotos, las añadimos
                if(diaSel && fotosSesion.length > 0) {
                    if(!db.apuntes) db.apuntes = {};
                    db.apuntes[diaSel] = (db.apuntes[diaSel] || []).concat(fotosSesion);
                    fotosSesion = [];
                }

                // 4. Subimos todo a la nube
                await enviarANube();
                alert("¡Datos publicados con éxito!");
            } catch(e) {
                alert("Error al sincronizar. Revisa tu conexión.");
            } finally {
                btn.innerText = "Sincronizar y Publicar";
                btn.disabled = false;
            }
        }

        async function eliminarPrueba(id) {
            if(!confirm("¿Borrar esta evaluación?")) return;
            db.pruebas = db.pruebas.filter(p => p.id !== id);
            await enviarANube();
        }

        async function eliminarFoto(fecha, index) {
            if(!confirm("¿Borrar esta foto?")) return;
            db.apuntes[fecha].splice(index, 1);
            await enviarANube();
        }

        async function enviarANube() {
            try {
                await fetch(`https://api.jsonbin.io/v3/b/${BIN_ID}`, {
                    method: 'PUT',
                    headers: { 'Content-Type': 'application/json', 'X-Master-Key': MASTER_KEY },
                    body: JSON.stringify(db)
                });
                actualizarPantalla();
            } catch(e) { alert("Error al conectar con la nube."); }
        }

        async function descargarDatos() {
            try {
                const res = await fetch(`https://api.jsonbin.io/v3/b/${BIN_ID}/latest`, {
                    headers: { 'X-Master-Key': MASTER_KEY, 'X-Bin-Meta': false }
                });
                db = await res.json();
                actualizarPantalla();
            } catch(e) {}
        }

        function actualizarPantalla() {
            document.querySelectorAll('.hay-prueba').forEach(el => el.classList.remove('hay-prueba'));
            
            // Render de Pruebas
            const divP = document.getElementById('lista-pruebas');
            divP.innerHTML = (db.pruebas || []).sort((a,b)=>new Date(a.f)-new Date(b.f)).map(p => {
                const celda = document.querySelector(`[data-date="${p.f}"]`);
                if(celda) celda.classList.add('hay-prueba');
                return `<div class="bg-white p-3 rounded-xl border border-slate-100 flex justify-between items-center shadow-sm">
                    <div class="flex flex-col">
                        <span class="text-xs font-bold text-slate-700 uppercase tracking-tighter">${p.n}</span>
                        <span class="text-[9px] font-black text-red-500">${p.f}</span>
                    </div>
                    <button onclick="eliminarPrueba(${p.id})" class="admin-only bg-red-50 text-red-500 w-7 h-7 rounded-full items-center justify-center text-[10px] font-bold">✕</button>
                </div>`;
            }).join('') || '<p class="text-[10px] text-slate-300 italic px-2">No hay evaluaciones programadas.</p>';

            // Render de Apuntes
            const divF = document.getElementById('lista-fotos');
            if(diaSel) {
                const fotos = (db.apuntes && db.apuntes[diaSel]) ? db.apuntes[diaSel] : [];
                divF.innerHTML = fotos.map((f, index) => `
                    <div class="bg-white rounded-2xl shadow-sm border border-slate-100 overflow-hidden relative mb-4">
                        <div class="p-2 bg-slate-50 flex justify-between items-center border-b">
                            <span class="text-[9px] font-bold text-blue-600 uppercase italic">${f.m}</span>
                            <button onclick="eliminarFoto('${diaSel}', ${index})" class="admin-only text-red-400 font-bold text-[9px] uppercase">Borrar Foto</button>
                        </div>
                        <img src="${f.i}" class="w-full">
                    </div>
                `).join('') || '<p class="text-[10px] text-slate-300 italic text-center py-10">Toca un día para ver sus apuntes.</p>';
            }
        }

        document.addEventListener('DOMContentLoaded', () => {
            const calendar = new FullCalendar.Calendar(document.getElementById('calendar'), {
                initialView: 'dayGridMonth', locale: 'es',
                headerToolbar: { left: 'prev', center: 'title', right: 'next' },
                dateClick: (info) => {
                    diaSel = info.dateStr;
                    document.getElementById('txt-fecha').innerText = "Viendo el día: " + diaSel;
                    actualizarPantalla();
                },
                datesSet: () => { setTimeout(actualizarPantalla, 300); }
            });
            calendar.render();
            descargarDatos();
            setInterval(descargarDatos, 60000); // Sincroniza cada 1 minuto
        });
    </script>
</body>
</html>
