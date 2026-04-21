¡Excelente elección de stack! Como arquitecto de software, te guiaré en la construcción de este sistema. Vamos a integrar la potencia de **Flutter** y **Firebase** bajo la metodología de **Antigravity**, que es un marco de trabajo diseñado para organizar la lógica de agentes y procesos de manera eficiente.

---

## ⚠️ Nota sobre Antigravity
Para efectos de esta práctica guiada, entenderemos **Antigravity** como un patrón de arquitectura de "Agentes y Tareas". En este modelo, no solo escribimos funciones, sino que definimos **Agentes** (clases especializadas) que poseen **Skills** (métodos CRUD) para cumplir un **Flujo de Trabajo**.

---

## Fase 1: Preparación y Configuración (Consola y Local)

### 1. Creación del Proyecto
En tu terminal, ejecuta:
```bash
flutter create crudropa
cd crudropa
```

### 2. Configuración en Firebase Console
1. Ve a [Firebase Console](https://console.firebase.google.com/).
2. Crea un proyecto llamado `crudropa-db`.
3. Registra una aplicación Android/iOS.
4. **Habilita Firestore Database**: Ve a "Cloud Firestore" -> "Crear base de datos" -> "Iniciar en modo de prueba".
5. Crea una colección llamada `productos`.

### 3. Integración de Librerías (`pubspec.yaml`)
Agregamos `firebase_core` para la conexión y `cloud_firestore` para el CRUD.

```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.0.0 # Revisa la versión más reciente
  cloud_firestore: ^5.0.0
```
*Para instalar, corre `flutter pub get` en la terminal.*

---

## Fase 2: Metodología Antigravity (Agentes y Roles)

En esta práctica, dividiremos el código según la lógica de agentes:

* **Agente:** `InventoryAgent` (Responsable de la persistencia).
* **Rol:** `DatabaseManager`.
* **Skills:** `CreateProduct`, `ReadProducts`, `UpdateProduct`, `DeleteProduct`.
* **Flujo:** `User Interaction` -> `Agent Skill` -> `Firestore Update`.

### Estructura de Carpetas Sugerida
```text
crudropa/
├── lib/
│   ├── agents/
│   │   └── inventory_agent.dart   # El "cerebro" que maneja la lógica
│   ├── models/
│   │   └── product_model.dart     # Estructura del dato
│   ├── ui/
│   │   └── inventory_screen.dart  # Interfaz de usuario
│   └── main.dart                  # Punto de entrada e inicialización
└── pubspec.yaml
```

---

## Fase 3: Implementación de Código Funcional

### 1. El Modelo de Datos (`models/product_model.dart`)
```dart
class Product {
  String id;
  String nombre;
  double precio;
  String categoria;

  Product({required this.id, required this.nombre, required this.precio, required this.categoria});

  // Convertir de Firestore a Objeto
  factory Product.fromFirestore(Map<String, dynamic> data, String id) {
    return Product(
      id: id,
      nombre: data['nombre'] ?? '',
      precio: (data['precio'] ?? 0.0).toDouble(),
      categoria: data['categoria'] ?? '',
    );
  }

  // Convertir de Objeto a Map para Firestore
  Map<String, dynamic> toMap() {
    return {
      'nombre': nombre,
      'precio': precio,
      'categoria': categoria,
    };
  }
}
```

### 2. El Agente con Skills CRUD (`agents/inventory_agent.dart`)
Aquí aplicamos el rol de `DatabaseManager`.

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/product_model.dart';

class InventoryAgent {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  final String collection = "productos";

  // Skill: Create
  Future<void> createProduct(String name, double price, String cat) async {
    await _db.collection(collection).add({
      'nombre': name,
      'precio': price,
      'categoria': cat,
    });
  }

  // Skill: Read (Stream para tiempo real)
  Stream<List<Product>> getProducts() {
    return _db.collection(collection).snapshots().map((snapshot) =>
        snapshot.docs.map((doc) => Product.fromFirestore(doc.data(), doc.id)).toList());
  }

  // Skill: Update
  Future<void> updateProduct(String id, String name, double price, String cat) async {
    await _db.collection(collection).doc(id).update({
      'nombre': name,
      'precio': price,
      'categoria': cat,
    });
  }

  // Skill: Delete
  Future<void> deleteProduct(String id) async {
    await _db.collection(collection).doc(id).delete();
  }
}
```

### 3. La Interfaz y Flujo de Trabajo (`ui/inventory_screen.dart`)
Este archivo orquesta la interacción del usuario con el Agente.

```dart
import 'package:flutter/material.dart';
import '../agents/inventory_agent.dart';
import '../models/product_model.dart';

class InventoryScreen extends StatelessWidget {
  final InventoryAgent agent = InventoryAgent();

  void _showForm(BuildContext context, {Product? product}) {
    final nameController = TextEditingController(text: product?.nombre);
    final priceController = TextEditingController(text: product?.precio.toString());
    final catController = TextEditingController(text: product?.categoria);

    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (_) => Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom, left: 15, right: 15, top: 15),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(controller: nameController, decoration: InputDecoration(labelText: 'Nombre')),
            TextField(controller: priceController, decoration: InputDecoration(labelText: 'Precio'), keyboardType: TextInputType.number),
            TextField(controller: catController, decoration: InputDecoration(labelText: 'Categoría')),
            ElevatedButton(
              onPressed: () {
                if (product == null) {
                  agent.createProduct(nameController.text, double.parse(priceController.text), catController.text);
                } else {
                  agent.updateProduct(product.id, nameController.text, double.parse(priceController.text), catController.text);
                }
                Navigator.pop(context);
              },
              child: Text(product == null ? 'Crear' : 'Actualizar'),
            )
          ],
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("CRUD Ropa - Agente Antigravity")),
      body: StreamBuilder<List<Product>>(
        stream: agent.getProducts(),
        builder: (context, snapshot) {
          if (!snapshot.hasData) return Center(child: CircularProgressIndicator());
          return ListView.builder(
            itemCount: snapshot.data!.length,
            itemBuilder: (context, index) {
              final p = snapshot.data![index];
              return ListTile(
                title: Text(p.nombre),
                subtitle: Text("${p.categoria} - \$${p.precio}"),
                trailing: Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    IconButton(icon: Icon(Icons.edit), onPressed: () => _showForm(context, product: p)),
                    IconButton(icon: Icon(Icons.delete), onPressed: () => agent.deleteProduct(p.id)),
                  ],
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showForm(context),
        child: Icon(Icons.add),
      ),
    );
  }
}
```

---

## Fase 4: Inicialización (`main.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'ui/inventory_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // Inicialización vital
  runApp(MaterialApp(
    home: InventoryScreen(),
    theme: ThemeData(primarySwatch: Colors.indigo),
  ));
}
```

### Resumen para Estudiantes:
1.  **Agente:** El `InventoryAgent` centraliza el poder.
2.  **Roles:** Al llamar a sus métodos, el agente asume la responsabilidad de hablar con la nube.
3.  **Skills:** Cada función (Add, Read, Update, Delete) es una habilidad específica.
4.  **Flujo:** El UI pide una acción -> El Agente ejecuta su Skill -> Firestore responde -> El UI se actualiza solo gracias al `StreamBuilder`.
