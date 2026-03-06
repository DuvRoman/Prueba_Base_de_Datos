# Prueba_Base_de_Datos

import xlsx from 'xlsx';
import Product from '../models/Product.js';
import Supplier from '../models/Supplier.js';
import ProductDetail from '../models/ProductDetail.js';

// Ahora recibimos 'filePath' (la ruta absoluta) en vez de 'fileBuffer'
export const processCoffeeExcel = async (filePath) => {
    
    // 1. LEER DESDE EL DISCO (Corregido)
    const workbook = xlsx.readFile(filePath); 
    const sheetName = workbook.SheetNames[0];
    const data = xlsx.utils.sheet_to_json(workbook.Sheets[sheetName]);

    console.log(`📊 Se encontraron ${data.length} filas en el Excel.`);
    let count = 0;

    for (const row of data) {
        try {
            // Si la fila está vacía o no tiene producto, la saltamos
            if (!row.PRODUCTO) continue; 

            // 2. EXTRAER DATOS DEL PROVEEDOR (De la columna PROVEEDOR_INFO)
            // Ejemplo de tu Excel: "Finca La Esperanza - Huila - NIT 900123"
            let sName = "Desconocido", sCity = "N/A", sNit = "000";

            if (row.PROVEEDOR_INFO && typeof row.PROVEEDOR_INFO === 'string') {
                const parts = row.PROVEEDOR_INFO.split('-');
                sName = parts[0] ? parts[0].trim() : "Desconocido";
                sCity = parts[1] ? parts[1].trim() : "N/A";
                // Limpiamos la palabra "NIT" para dejar solo el número
                sNit = parts[2] ? parts[2].replace(/NIT/i, '').trim() : "S/N";
            }

            // A. SQL: Guardar Proveedor (Usamos las variables extraídas)
            await Supplier.findOrCreate({
                where: { supplier_nit: sNit },
                defaults: { supplier_name: sName, supplier_city: sCity }
            });

            // B. SQL: Guardar Producto (Conectamos tu Excel con el Modelo)
            await Product.findOrCreate({
                where: { product_name: row.PRODUCTO }, // Viene de tu columna PRODUCTO
                defaults: {
                    unit_price: row.PRECIO || 0,       // Viene de tu columna PRECIO
                    quantity_stock: row.STOCK || 0,    // Viene de tu columna STOCK
                    product_category: 'Grano/Preparado'
                }
            });

            // C. NoSQL: Inicializar Detalles de Cata en MongoDB
            await ProductDetail.findOneAndUpdate(
                { product_name: row.PRODUCTO }, // Viene de tu columna PRODUCTO
                { 
                    $setOnInsert: { 
                        tasting_notes: ["Origen único", "Cosecha manual"],
                        roast_level: "Medio" 
                    } 
                },
                { upsert: true }
            );

            count++;
            console.log(`✅ ${row.PRODUCTO} guardado correctamente.`);

        } catch (error) {
            console.error(`❌ Error en fila ${row.PRODUCTO}:`, error.message);
        }
    }
    
    return { success: true, count: count, message: "Importación de datos de Café finalizada" };
};
