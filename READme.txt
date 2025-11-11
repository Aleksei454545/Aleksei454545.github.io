/index.php/

<?php
header('Content-Type: application/json; charset=utf-8');              // говорим браузеру: ответ будет JSON в UTF-8
header('Access-Control-Allow-Origin: *');                              // разрешаем доступ со всех доменов (CORS)
header('Access-Control-Allow-Methods: GET, OPTIONS');                  // разрешены методы GET и OPTIONS
header('Access-Control-Allow-Headers: Content-Type');                  // разрешаем заголовок Content-Type
if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') { http_response_code(204); exit; } // на префлайт-запрос OPTIONS сразу отвечаем 204 и выходим

require __DIR__ . '/db.php';                                           // подключаем файл с функцией db(), которая возвращает PDO

function respond(array $arr) {                                         // вспомогательная функция для ответа в JSON
  $flags = JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES;            // не экранировать Юникод и слэши
  if (isset($_GET['pretty'])) $flags |= JSON_PRETTY_PRINT;             // если ?pretty — красиво форматируем JSON
  echo json_encode($arr, $flags);                                      // отправляем JSON
  exit;                                                                 // и останавливаем скрипт
}

try {                                                                   // ловим любые ошибки ниже
  $path = $_GET['r'] ?? '';                                            // читаем параметр маршрута r из URL
  if ($path === 'categories') listCategories();                        // r=categories → вызвать listCategories()
  elseif ($path === 'products') listProducts();                        // r=products   → вызвать listProducts()
  elseif ($path === 'product') getProduct();                           // r=product    → вызвать getProduct()
  else respond(['ok'=>false, 'error'=>'Unknown route']);               // иначе: неизвестный маршрут
} catch (Throwable $e) {                                               // если случилась ошибка
  http_response_code(500);                                             // ставим код 500 (внутренняя ошибка сервера)
  respond(['ok'=>false, 'error'=>$e->getMessage()]);                   // отправляем JSON с текстом ошибки
}

function listCategories() {                                            // обработчик: список категорий
  $pdo = db();                                                         // берём подключение PDO
  $rows = $pdo->query("SELECT id, slug, name FROM categories ORDER BY name")->fetchAll(); // выбираем id/slug/name всех категорий
  respond(['ok'=>true, 'data'=>$rows]);                                // отдаём { ok:true, data:[...] }
}

function listProducts() {                                              // обработчик: список товаров (каталог) с фильтрами
  $pdo = db();                                                         // PDO

  $category = $_GET['category'] ?? null;                               // фильтр по категории (slug)
  $brand    = $_GET['brand'] ?? null;                                  // фильтр по бренду
  $q        = $_GET['q'] ?? null;                                      // поиск по названию
  $min      = $_GET['min'] ?? null;                                    // минимальная цена
  $max      = $_GET['max'] ?? null;                                    // максимальная цена
  $limit    = (int)($_GET['limit'] ?? 20);                             // сколько записей вернуть
  $offset   = (int)($_GET['offset'] ?? 0);                             // с какого смещения начать (пагинация)
  if ($limit < 1 || $limit > 100) $limit = 20;                         // ограничиваем лимит 1…100
  if ($offset < 0) $offset = 0;                                        // отрицательное смещение не допускаем

  $where = []; $params = [];                                           // будем накапливать условия WHERE и параметры
  if ($category) {                                                     // если задан slug категории
    $where[] = "p.category_id = (SELECT id FROM categories WHERE slug = :slug)"; // фильтруем по категории через подзапрос
    $params[':slug'] = $category;                                      // параметр :slug
  }
  if ($brand)    { $where[] = "p.brand = :brand"; $params[':brand'] = $brand; }  // фильтр по бренду
  if ($q)        { $where[] = "p.title LIKE :q"; $params[':q'] = '%'.$q.'%'; }   // поиск по названию (LIKE %q%)
  if ($min !== null) { $where[] = "p.price >= :minp"; $params[':minp'] = (float)$min; } // цена от
  if ($max !== null) { $where[] = "p.price <= :maxp"; $params[':maxp'] = (float)$max; } // цена до
  $wsql = $where ? 'WHERE '.implode(' AND ', $where) : '';             // собираем WHERE … AND … или пусто

  $sql = "
    SELECT p.id, c.slug AS category, p.title, p.brand, p.price, p.image_url, p.short_desc
    FROM products p
    JOIN categories c ON c.id = p.category_id
    $wsql
    ORDER BY p.id DESC
    LIMIT :limit OFFSET :offset
  ";                                                                   // основной запрос: берём поля для карточек + slug категории
  $stmt = $pdo->prepare($sql);                                         // готовим запрос
  foreach ($params as $k=>$v) $stmt->bindValue($k,$v);                 // подставляем все параметры фильтра
  $stmt->bindValue(':limit',$limit,PDO::PARAM_INT);                    // подставляем лимит (как int)
  $stmt->bindValue(':offset',$offset,PDO::PARAM_INT);                  // подставляем смещение (как int)
  $stmt->execute();                                                    // выполняем запрос
  $rows = $stmt->fetchAll();                                           // получаем все строки товаров

  $stmt2 = $pdo->prepare("SELECT COUNT(*) cnt FROM products p JOIN categories c ON c.id=p.category_id $wsql"); // второй запрос: посчитать всего
  foreach ($params as $k=>$v) $stmt2->bindValue($k,$v);                // те же параметры WHERE
  $stmt2->execute();                                                   // выполняем
  $total = (int)$stmt2->fetch()['cnt'];                                // достаём число из колонки cnt

  respond(['ok'=>true, 'data'=>$rows, 'total'=>$total, 'limit'=>$limit, 'offset'=>$offset]); // отдаём товары и инфо для пагинации
}

function getProduct() {                                                // обработчик: один товар по id
  $pdo = db();                                                         // PDO
  $id = (int)($_GET['id'] ?? 0);                                       // берём id из URL
  if ($id <= 0) respond(['ok'=>false, 'error'=>'Bad id']);             // если id некорректный — сразу ошибка

  $stmt = $pdo->prepare("
    SELECT p.id, c.slug AS category, p.title, p.brand, p.price, p.image_url, p.short_desc, p.created_at
    FROM products p
    JOIN categories c ON c.id = p.category_id
    WHERE p.id = :id
    LIMIT 1
  ");                                                                  // запрос основной информации по товару + slug категории
  $stmt->execute([':id' => $id]);                                      // подставляем :id
  $product = $stmt->fetch();                                           // берём строку
  if (!$product) respond(['ok'=>false, 'error'=>'Not found']);         // если не нашли — отвечаем ошибкой

  $specsSql = "
    WITH ranked AS (                                                    -- CTE: временная таблица ranked
      SELECT
        spec_name, spec_value, unit, value_num,                         -- поля характеристики
        ROW_NUMBER() OVER (PARTITION BY LOWER(spec_name)                -- нумеруем строки по имени характеристики (без регистра)
                              ORDER BY id DESC) AS rn                    -- последние по id будут с rn=1
      FROM product_specs
      WHERE product_id = :id_specs                                      -- только для текущего товара
    )
    SELECT spec_name, spec_value, unit, value_num                        -- берём
    FROM ranked
    WHERE rn = 1                                                         -- оставляем по одной (самой новой) записи на spec_name
    ORDER BY spec_name                                                   -- сортируем по имени характеристики
  ";                                                                     // ⚠ нужно MySQL 8.0+ (или MariaDB с окнами/CTE)
  $specs = $pdo->prepare($specsSql);                                     // готовим запрос на характеристики
  $specs->execute([':id_specs' => $id]);                                  // подставляем id товара
  $product['specs'] = $specs->fetchAll();                                 // добавляем массив specs внутрь $product

  respond(['ok'=>true, 'data'=>$product]);                                // отдаём товар с характеристиками
}


