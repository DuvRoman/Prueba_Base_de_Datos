# Prueba_Base_de_Datos

import xlsx from 'xlsx';
import fs from 'fs';
import Product from '../models/Product.js';
import Supplier from '../models/Supplier.js';
import ProductDetail from '../models/ProductDetail.js';

export const processCoffeeExcel = async (filePath) => {
    try {
        // 1. VERIFICAR SI EL ARCHIVO EXISTE EN UPLOADS
        if (!fs.existsSync(filePath)) {
            throw new Error(`El archivo no se encuentra en la ruta: ${filePath}`);
        }

        // 2. LEER DESDE DISCO (Usando readFile)
        const workbook = xlsx.readFile(filePath); 
        const sheetName = workbook.SheetNames[0];
        const data = xlsx.utils.sheet_to_json(workbook.Sheets[sheetName]);

        console.log(`📊 Filas detectadas en el Excel: ${data.length}`);

        let count = 0;

        for (const row of data) {
            try {
                // Validación: Si la fila no tiene nombre de producto, la saltamos
                if (!row.PRODUCTO) continue;

                // --- PROCESAR PROVEEDOR_INFO ---
                let sName = "Interno", sCity = "N/A", sNit = "000";
                if (row.PROVEEDOR_INFO && row.PROVEEDOR_INFO !== "Interno (Preparado)") {
                    const parts = String(row.PROVEEDOR_INFO).split('-');
                    sName = parts[0]?.trim() || "Desconocido";
                    sCity = parts[1]?.trim() || "N/A";
                    sNit = parts[2] ? parts[2].replace(/NIT/i, '').trim() : "S/N";
                }

                // A. SQL: Guardar Proveedor
                await Supplier.findOrCreate({
                    where: { supplier_nit: String(sNit) },
                    defaults: { supplier_name: sName, supplier_city: sCity }
                });

                // B. SQL: Guardar Producto (Usando los nombres exactos de tu imagen)
                await Product.findOrCreate({
                    where: { product_name: String(row.PRODUCTO).trim() },
                    defaults: {
                        unit_price: parseFloat(row.PRECIO) || 0,
                        quantity_stock: parseInt(row.STOCK) || 0,
                        product_category: 'Café de Origen'
                    }
                });

                // C. NoSQL: MongoDB (Detalles de Cata)
                await ProductDetail.findOneAndUpdate(
                    { product_name: String(row.PRODUCTO).trim() },
                    { 
                        $setOnInsert: { 
                            tasting_notes: ["Origen único", "Cosecha manual"],
                            roast_level: "Medio",
                            last_update: new Date()
                        } 
                    },
                    { upsert: true }
                );

                count++;
                console.log(`✅ Procesado con éxito: ${row.PRODUCTO}`);

            } catch (rowError) {
                console.error(`❌ Error en fila ${row.PRODUCTO}:`, rowError.message);
            }
        }

        return { success: true, count };

    } catch (error) {
        console.error("❌ Error fatal en el servicio:", error.message);
        throw error;
    }
};
