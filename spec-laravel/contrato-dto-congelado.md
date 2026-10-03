# Contrato DTO Congelado — Simple Stock Flow

> **Fuente congelada de verdad para frontend y backend.**  
> Todo DTO en Laravel (`app/Presentation/`) y en React (`src/infrastructure/http/dto/api.dto.ts`) debe reflejar exactamente estos tipos y nombres en camelCase.

---

## 1. Convenciones Globales

* **Formato de datos:** `application/json` (salvo `/api/products/{id}/image` que es `multipart/form-data` y `/media/{key}` que es binario de imagen).
* **Nombres de atributos:** `camelCase` estricto en JSON.
* **Moneda:** Cantidades en números decimales con 2 posiciones (ej. `12500.00`). No cadenas con formato de moneda. Moneda por defecto `COP`.
* **Fechas:** ISO-8601 UTC estricto con sufijo `Z` o `+00:00` (ej. `2026-10-03T14:00:00+00:00`).
* **Paginación:** Objetos paginados devuelven `{ "items": [...], "total": 45, "page": 1, "pageSize": 20, "totalPages": 3 }`.

---

## 2. Los 3 Formatos de Error

### A. Regla de Negocio / Conflicto / Error Interno (422, 409, 500)
`Content-Type: application/problem+json`
```json
{
  "type": "about:blank",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "El stock disponible (3) es insuficiente para la cantidad solicitada (5)"
}
```
*El campo `detail` es obligatorio y redactado en español legible.*

### B. Formato de Petición Inválido (400 Bad Request)
`Content-Type: application/problem+json`
```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "La solicitud contiene errores de validación de formato",
  "errors": {
    "username": ["El campo username es obligatorio y debe ser una cadena."],
    "lines": ["El campo lines debe ser una lista no vacía."]
  }
}
```

### C. Autenticación, Permisos, No Encontrado, Método No Permitido (401, 403, 404, 405)
* **Cuerpo:** Vacío.
* **Content-Length:** `0`
* **Headers obligatorios:** `WWW-Authenticate: Bearer` en 401; `Allow: GET, POST...` en 405.

---

## 3. Especificación Campo por Campo de los 15 Endpoints

### E-01 · POST `/api/auth/login`
* **Auth:** Anónimo
* **Request:**
  ```json
  {
    "username": "admin",
    "password": "SecretPassword123!"
  }
  ```
* **Response (200 OK):**
  ```json
  {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "c1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c",
      "username": "admin",
      "role": "admin"
    }
  }
  ```

### E-02 · POST `/api/auth/register`
* **Auth:** Solo `admin` (Bearer Token)
* **Request:**
  ```json
  {
    "username": "carlos_vendedor",
    "password": "SellerPassword123!"
  }
  ```
  *(Por regla DP-04, no se permite crear admins; solo se crean usuarios con rol `seller`).*
* **Response (201 Created):**
  ```json
  {
    "id": "e2b3c4d5-e6f7-8a9b-0c1d-2e3f4a5b6c7d",
    "username": "carlos_vendedor",
    "role": "seller"
  }
  ```

### E-03 · GET `/api/products`
* **Auth:** Autenticado (`admin` o `seller`)
* **Query Params:**
  * `search` (opcional): string
  * `categoryId` (opcional): UUID
  * `page` (opcional, default: 1): integer
  * `pageSize` (opcional, default: 20, max: 100): integer
* **Response (200 OK):**
  ```json
  {
    "items": [
      {
        "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "name": "Café Colombiano Especial",
        "price": 18500.00,
        "currency": "COP",
        "stock": 15,
        "categoryId": "2fa85f64-5717-4562-b3fc-2c963f66afa5",
        "categoryName": "Bebidas",
        "imageUrl": "http://localhost:8000/media/img_cafe_01.jpg"
      }
    ],
    "total": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1
  }
  ```

### E-04 · GET `/api/products/{id}`
* **Auth:** Autenticado
* **Response (200 OK):** DTO individual de producto igual al item de la lista.

### E-05 · POST `/api/products`
* **Auth:** Solo `admin`
* **Request:**
  ```json
  {
    "name": "Café Colombiano Especial",
    "price": 18500.00,
    "currency": "COP",
    "initialStock": 20,
    "categoryId": "2fa85f64-5717-4562-b3fc-2c963f66afa5"
  }
  ```
* **Response (201 Created):** DTO de producto creado.

### E-06 · PUT `/api/products/{id}`
* **Auth:** Solo `admin`
* **Request:**
  ```json
  {
    "name": "Café Colombiano Supremo",
    "price": 19500.00,
    "currency": "COP",
    "categoryId": "2fa85f64-5717-4562-b3fc-2c963f66afa5"
  }
  ```
  *(El stock no se altera por este endpoint; solo a través de ventas o ajustes).*
* **Response (200 OK):** DTO de producto actualizado.

### E-07 · DELETE `/api/products/{id}`
* **Auth:** Solo `admin`
* **Response (204 No Content):** Cuerpo vacío. (Baja lógica, nunca borrado físico si hubo ventas).

### E-08 · POST `/api/products/{id}/image`
* **Auth:** Solo `admin`
* **Content-Type:** `multipart/form-data` con campo `file`
* **Response (200 OK):**
  ```json
  {
    "imageUrl": "http://localhost:8000/media/9f8c406a-img.png"
  }
  ```

### E-09 · GET `/api/categories`
* **Auth:** Autenticado
* **Response (200 OK):**
  ```json
  [
    {
      "id": "2fa85f64-5717-4562-b3fc-2c963f66afa5",
      "name": "Bebidas"
    },
    {
      "id": "4fa85f64-5717-4562-b3fc-2c963f66afa7",
      "name": "Snacks"
    }
  ]
  ```

### E-10 · POST `/api/sales`
* **Auth:** Autenticado
* **Request:**
  ```json
  {
    "items": [
      {
        "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "quantity": 2
      }
    ]
  }
  ```
* **Response (201 Created):**
  ```json
  {
    "id": "7fa85f64-5717-4562-b3fc-2c963f66afa9",
    "sellerId": "c1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c",
    "sellerUsername": "admin",
    "createdAt": "2026-10-03T14:15:00+00:00",
    "total": 37000.00,
    "currency": "COP",
    "items": [
      {
        "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "productName": "Café Colombiano Especial",
        "unitPrice": 18500.00,
        "quantity": 2,
        "subtotal": 37000.00
      }
    ]
  }
  ```

### E-11 · GET `/api/sales`
* **Auth:** Autenticado
* **Query Params:** `page`, `pageSize`, `from` (ISO), `to` (ISO)
* **Response (200 OK):** Paginación de ventas resumidas con totales.

### E-12 · GET `/api/sales/{id}`
* **Auth:** Autenticado
* **Response (200 OK):** Detalle de venta completo con sus líneas congeladas.

### E-13 · GET `/api/reports/sales?from=2026-10-01T00:00:00Z&to=2026-10-04T00:00:00Z`
* **Auth:** Autenticado
* **Response (200 OK):**
  ```json
  {
    "from": "2026-10-01T00:00:00+00:00",
    "to": "2026-10-04T00:00:00+00:00",
    "totalAmount": 150000.00,
    "currency": "COP",
    "items": [
      {
        "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "productName": "Café Colombiano Supremo",
        "unitsSold": 8,
        "totalAmount": 150000.00
      }
    ]
  }
  ```

### E-14 · GET `/health`
* **Auth:** Anónimo
* **Response (200 OK):**
  ```json
  {
    "status": "pass",
    "database": "connected"
  }
  ```

### E-15 · GET `/media/{key}`
* **Auth:** Anónimo
* **Response (200 OK):** Binario con `Content-Type: image/png` o `image/jpeg`.
