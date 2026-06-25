# Por-si-las-dud



<?php
session_start();

// Configuración
define('DB_FILE', 'db.txt');
define('UPLOAD_DIR', 'uploads/');

// Crear directorio de uploads si no existe
if (!file_exists(UPLOAD_DIR)) {
    mkdir(UPLOAD_DIR, 0777, true);
}

// Funciones de base de datos
function loadDB() {
    if (!file_exists(DB_FILE)) {
        file_put_contents(DB_FILE, json_encode(['users' => [], 'files' => []]));
    }
    return json_decode(file_get_contents(DB_FILE), true);
}

function saveDB($data) {
    file_put_contents(DB_FILE, json_encode($data, JSON_PRETTY_PRINT));
}

// Procesar acciones
$action = $_GET['action'] ?? $_POST['action'] ?? '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $db = loadDB();
    
    // Registro
    if ($action === 'register') {
        $username = trim($_POST['username']);
        $password = $_POST['password'];
        
        if (isset($db['users'][$username])) {
            $error = "El usuario ya existe";
        } else {
            $db['users'][$username] = [
                'password' => password_hash($password, PASSWORD_DEFAULT),
                'created_at' => date('Y-m-d H:i:s')
            ];
            saveDB($db);
            $_SESSION['user'] = $username;
            
            // Crear carpeta del usuario
            $userDir = UPLOAD_DIR . $username;
            if (!file_exists($userDir)) {
                mkdir($userDir, 0777, true);
            }
            
            header('Location: index.php');
            exit;
        }
    }
    
    // Login
    if ($action === 'login') {
        $username = trim($_POST['username']);
        $password = $_POST['password'];
        
        if (!isset($db['users'][$username]) || !password_verify($password, $db['users'][$username]['password'])) {
            $error = "Usuario o contraseña incorrectos";
        } else {
            $_SESSION['user'] = $username;
            header('Location: index.php');
            exit;
        }
    }
    
    // Subir archivo
    if ($action === 'upload' && isset($_SESSION['user'])) {
        $file = $_FILES['file'];
        $isPublic = isset($_POST['is_public']) ? true : false;
        
        if ($file['error'] === UPLOAD_ERR_OK) {
            $username = $_SESSION['user'];
            $userDir = UPLOAD_DIR . $username;
            
            if (!file_exists($userDir)) {
                mkdir($userDir, 0777, true);
            }
            
            $originalName = $file['name'];
            $fileExt = pathinfo($originalName, PATHINFO_EXTENSION);
            $newFileName = uniqid() . '_' . date('Y-m-d_H-i-s') . '.' . $fileExt;
            $destination = $userDir . '/' . $newFileName;
            
            if (move_uploaded_file($file['tmp_name'], $destination)) {
                $db['files'][] = [
                    'id' => uniqid(),
                    'user' => $username,
                    'original_name' => $originalName,
                    'stored_name' => $newFileName,
                    'path' => $destination,
                    'size' => $file['size'],
                    'type' => $file['type'],
                    'is_public' => $isPublic,
                    'upload_date' => date('Y-m-d H:i:s')
                ];
                saveDB($db);
                $success = "Archivo subido exitosamente";
            } else {
                $error = "Error al subir el archivo";
            }
        } else {
            $error = "Error en la carga del archivo";
        }
    }
    
    // Cambiar visibilidad
    if ($action === 'toggle_visibility' && isset($_POST['file_id']) && isset($_SESSION['user'])) {
        $fileId = $_POST['file_id'];
        foreach ($db['files'] as &$file) {
            if ($file['id'] === $fileId && $file['user'] === $_SESSION['user']) {
                $file['is_public'] = !$file['is_public'];
                saveDB($db);
                break;
            }
        }
        header('Location: index.php');
        exit;
    }
}

// Logout
if ($action === 'logout') {
    session_destroy();
    header('Location: index.php');
    exit;
}

// Obtener datos actualizados
$db = loadDB();
?>

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sistema de Gestión de Archivos</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }

        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            overflow: hidden;
        }

        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            text-align: center;
        }

        .header h1 {
            font-size: 2.5em;
            margin-bottom: 10px;
            animation: slideDown 0.5s ease;
        }

        @keyframes slideDown {
            from {
                transform: translateY(-50px);
                opacity: 0;
            }
            to {
                transform: translateY(0);
                opacity: 1;
            }
        }

        .nav {
            background: #f8f9fa;
            padding: 15px 30px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid #dee2e6;
        }

        .nav-tabs {
            display: flex;
            gap: 15px;
        }

        .nav-tab {
            padding: 10px 20px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: all 0.3s ease;
            text-decoration: none;
            display: inline-block;
        }

        .nav-tab:hover {
            background: #0056b3;
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(0,0,0,0.3);
        }

        .nav-tab.active {
            background: #28a745;
        }

        .content {
            padding: 30px;
        }

        .form-container {
            max-width: 400px;
            margin: 0 auto;
            padding: 30px;
            background: #f8f9fa;
            border-radius: 10px;
            box-shadow: 0 5px 15px rgba(0,0,0,0.1);
        }

        .form-group {
            margin-bottom: 20px;
        }

        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
            color: #333;
        }

        .form-group input[type="text"],
        .form-group input[type="password"],
        .form-group input[type="file"] {
            width: 100%;
            padding: 10px;
            border: 2px solid #dee2e6;
            border-radius: 5px;
            transition: border-color 0.3s ease;
        }

        .form-group input:focus {
            outline: none;
            border-color: #667eea;
        }

        .btn {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 12px 30px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 16px;
            transition: all 0.3s ease;
            width: 100%;
        }

        .btn:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(0,0,0,0.3);
        }

        .btn-small {
            padding: 8px 15px;
            font-size: 14px;
            width: auto;
        }

        .file-list {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
            gap: 20px;
            margin-top: 20px;
        }

        .file-card {
            background: white;
            border: 1px solid #dee2e6;
            border-radius: 10px;
            padding: 20px;
            transition: all 0.3s ease;
            animation: fadeIn 0.5s ease;
        }

        .file-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 25px rgba(0,0,0,0.1);
        }

        @keyframes fadeIn {
            from {
                opacity: 0;
                transform: translateY(20px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }

        .file-card h3 {
            color: #333;
            margin-bottom: 10px;
            word-break: break-all;
        }

        .file-info {
            color: #666;
            font-size: 0.9em;
            margin-bottom: 10px;
        }

        .badge {
            display: inline-block;
            padding: 5px 10px;
            border-radius: 20px;
            font-size: 0.8em;
            font-weight: bold;
            margin-right: 5px;
        }

        .badge-public {
            background: #28a745;
            color: white;
        }

        .badge-private {
            background: #dc3545;
            color: white;
        }

        .error {
            background: #f8d7da;
            color: #721c24;
            padding: 10px;
            border-radius: 5px;
            margin-bottom: 20px;
            border: 1px solid #f5c6cb;
        }

        .success {
            background: #d4edda;
            color: #155724;
            padding: 10px;
            border-radius: 5px;
            margin-bottom: 20px;
            border: 1px solid #c3e6cb;
        }

        .upload-animation {
            display: none;
            text-align: center;
            padding: 20px;
        }

        .upload-animation.active {
            display: block;
            animation: pulse 1s infinite;
        }

        @keyframes pulse {
            0% { transform: scale(1); }
            50% { transform: scale(1.1); }
            100% { transform: scale(1); }
        }

        .spinner {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #667eea;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
            margin: 20px auto;
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }

        .checkbox-group {
            display: flex;
            align-items: center;
            gap: 10px;
        }

        .checkbox-group input[type="checkbox"] {
            width: auto;
        }

        @media (max-width: 768px) {
            .file-list {
                grid-template-columns: 1fr;
            }
            
            .nav-tabs {
                flex-direction: column;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>📁 Sistema de Gestión de Archivos</h1>
            <?php if (isset($_SESSION['user'])): ?>
                <p>Bienvenido, <?php echo htmlspecialchars($_SESSION['user']); ?></p>
            <?php endif; ?>
        </div>

        <div class="nav">
            <div class="nav-tabs">
                <?php if (!isset($_SESSION['user'])): ?>
                    <a href="?tab=login" class="nav-tab <?php echo ($_GET['tab'] ?? '') === 'login' || !isset($_GET['tab']) ? 'active' : ''; ?>">Iniciar Sesión</a>
                    <a href="?tab=register" class="nav-tab <?php echo ($_GET['tab'] ?? '') === 'register' ? 'active' : ''; ?>">Registrarse</a>
                <?php else: ?>
                    <a href="?tab=upload" class="nav-tab <?php echo ($_GET['tab'] ?? '') === 'upload' || !isset($_GET['tab']) ? 'active' : ''; ?>">Subir Archivo</a>
                    <a href="?tab=myfiles" class="nav-tab <?php echo ($_GET['tab'] ?? '') === 'myfiles' ? 'active' : ''; ?>">Mis Archivos</a>
                    <a href="?tab=allfiles" class="nav-tab <?php echo ($_GET['tab'] ?? '') === 'allfiles' ? 'active' : ''; ?>">Archivos Públicos</a>
                    <a href="?action=logout" class="nav-tab" style="background: #dc3545;">Cerrar Sesión</a>
                <?php endif; ?>
            </div>
        </div>

        <div class="content">
            <?php if (isset($error)): ?>
                <div class="error"><?php echo $error; ?></div>
            <?php endif; ?>
            
            <?php if (isset($success)): ?>
                <div class="success"><?php echo $success; ?></div>
            <?php endif; ?>

            <?php
            $tab = $_GET['tab'] ?? '';

            if (!isset($_SESSION['user'])):
                // Mostrar formularios de login/registro
                if ($tab === 'register'): ?>
                    <div class="form-container">
                        <h2>Registrarse</h2>
                        <form method="POST" action="?action=register">
                            <input type="hidden" name="action" value="register">
                            <div class="form-group">
                                <label>Usuario:</label>
                                <input type="text" name="username" required>
                            </div>
                            <div class="form-group">
                                <label>Contraseña:</label>
                                <input type="password" name="password" required>
                            </div>
                            <button type="submit" class="btn">Registrarse</button>
                        </form>
                    </div>
                <?php else: ?>
                    <div class="form-container">
                        <h2>Iniciar Sesión</h2>
                        <form method="POST" action="?action=login">
                            <input type="hidden" name="action" value="login">
                            <div class="form-group">
                                <label>Usuario:</label>
                                <input type="text" name="username" required>
                            </div>
                            <div class="form-group">
                                <label>Contraseña:</label>
                                <input type="password" name="password" required>
                            </div>
                            <button type="submit" class="btn">Iniciar Sesión</button>
                        </form>
                    </div>
                <?php endif;

            else:
                // Usuario logueado
                if ($tab === 'myfiles'):
                    // Mostrar archivos del usuario
                    $userFiles = array_filter($db['files'], function($file) {
                        return $file['user'] === $_SESSION['user'];
                    });
                ?>
                    <h2>Mis Archivos</h2>
                    <div class="file-list">
                        <?php foreach ($userFiles as $file): ?>
                            <div class="file-card">
                                <h3><?php echo htmlspecialchars($file['original_name']); ?></h3>
                                <div class="file-info">
                                    <p>Fecha: <?php echo $file['upload_date']; ?></p>
                                    <p>Tamaño: <?php echo number_format($file['size'] / 1024, 2); ?> KB</p>
                                    <p>Tipo: <?php echo $file['type']; ?></p>
                                </div>
                                <span class="badge <?php echo $file['is_public'] ? 'badge-public' : 'badge-private'; ?>">
                                    <?php echo $file['is_public'] ? 'Público' : 'Privado'; ?>
                                </span>
                                <br><br>
                                <a href="<?php echo $file['path']; ?>" download class="btn btn-small" style="display: inline-block; margin-right: 5px;">Descargar</a>
                                <form method="POST" style="display: inline;">
                                    <input type="hidden" name="action" value="toggle_visibility">
                                    <input type="hidden" name="file_id" value="<?php echo $file['id']; ?>">
                                    <button type="submit" class="btn btn-small" style="background: <?php echo $file['is_public'] ? '#dc3545' : '#28a745'; ?>">
                                        <?php echo $file['is_public'] ? 'Hacer Privado' : 'Hacer Público'; ?>
                                    </button>
                                </form>
                            </div>
                        <?php endforeach; ?>
                        <?php if (empty($userFiles)): ?>
                            <p>No has subido ningún archivo aún.</p>
                        <?php endif; ?>
                    </div>

                <?php elseif ($tab === 'allfiles'): 
                    // Mostrar todos los archivos públicos
                    $publicFiles = array_filter($db['files'], function($file) {
                        return $file['is_public'];
                    });
                ?>
                    <h2>Archivos Públicos de Todos los Usuarios</h2>
                    <div class="file-list">
                        <?php foreach ($publicFiles as $file): ?>
                            <div class="file-card">
                                <h3><?php echo htmlspecialchars($file['original_name']); ?></h3>
                                <div class="file-info">
                                    <p>Subido por: <?php echo htmlspecialchars($file['user']); ?></p>
                                    <p>Fecha: <?php echo $file['upload_date']; ?></p>
                                    <p>Tamaño: <?php echo number_format($file['size'] / 1024, 2); ?> KB</p>
                                    <p>Tipo: <?php echo $file['type']; ?></p>
                                </div>
                                <a href="<?php echo $file['path']; ?>" target="_blank" class="btn btn-small" style="display: inline-block; margin-right: 5px; background: #17a2b8;">Visualizar</a>
                                <a href="<?php echo $file['path']; ?>" download class="btn btn-small" style="display: inline-block;">Descargar</a>
                            </div>
                        <?php endforeach; ?>
                        <?php if (empty($publicFiles)): ?>
                            <p>No hay archivos públicos disponibles.</p>
                        <?php endif; ?>
                    </div>

                <?php else: 
                    // Página de subida de archivos
                ?>
                    <div class="form-container">
                        <h2>Subir Archivo</h2>
                        <form method="POST" enctype="multipart/form-data" id="uploadForm">
                            <input type="hidden" name="action" value="upload">
                            <div class="form-group">
                                <label>Seleccionar archivo (todos los formatos permitidos):</label>
                                <input type="file" name="file" required>
                            </div>
                            <div class="form-group">
                                <div class="checkbox-group">
                                    <input type="checkbox" name="is_public" id="is_public" value="1">
                                    <label for="is_public">Hacer archivo público</label>
                                </div>
                            </div>
                            <button type="submit" class="btn" onclick="showUploadAnimation()">Subir Archivo</button>
                        </form>
                        <div class="upload-animation" id="uploadAnimation">
                            <div class="spinner"></div>
                            <p>Subiendo archivo...</p>
                        </div>
                    </div>
                <?php endif; ?>
            <?php endif; ?>
        </div>
    </div>

    <script>
        function showUploadAnimation() {
            document.getElementById('uploadAnimation').classList.add('active');
        }
    </script>
</body>
</html>
