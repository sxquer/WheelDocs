Мануал по использованию класса SyncCaseMonitoringCommand
Содержание

Введение
Описание класса
Конструктор и настройка
Основные методы
Вспомогательные методы
Константы
Примеры использования
Полный жизненный цикл
Возможные проблемы и их решения
Введение

Класс SyncCaseMonitoringCommand предназначен для синхронизации дел с системой мониторинга CyberJustice. Он обеспечивает актуальность списка дел, находящихся под мониторингом, в соответствии с данными из базы данных AmoCRM.

Описание класса

Copynamespace App\Console\Commands\CyberJustice;

use AmoCRM\Collections\TasksCollection;
use App\Domain\AmoAPI\AmoAPI;
use App\Domain\AmoAPI\AmoCFManager;
use App\Domain\AmoAPI\AmoHelper;
use App\Domain\AmoAPI\AmoTaskManager;
use App\Http\Controllers\Custom\CyberJustice\CyberJusticeCaseController;
use Illuminate\Console\Command;
use App\Services\CyberJusticeApiService;
use App\Models\AmoLead;
use App\Models\KadarbitrCase;
use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Redis;

class SyncCaseMonitoringCommand extends Command
{
    // ...
}
Класс расширяет базовый класс Command фреймворка Laravel и предназначен для запуска через консольный интерфейс Artisan.

Конструктор и настройка

Copyprotected $signature   = 'cyberjustice:sync:case-monitoring {--clear} {--test}';
protected $description = 'Синхронизация дел для мониторинга (CyberJustice).';

private const MONITORING_LIMIT   = 500;
private const RETRIEVE_REDIS_KEY = 'cyberjustice:case:retrieve';
private const OVERFLOW_REDIS_KEY = 'cyberjustice:case:overflow';

private $logger;

public function __construct()
{
    parent::__construct();
    $this->logger = Log::channel('cyber_justice');
}
Параметры запуска

--clear - флаг для полной очистки мониторинга и связанных ключей Redis
--test - флаг для запуска в тестовом режиме (без реального выполнения операций)
Константы

MONITORING_LIMIT - максимальное количество дел для мониторинга (500)
RETRIEVE_REDIS_KEY - ключ Redis для хранения номеров дел в статусе "accepted"
OVERFLOW_REDIS_KEY - ключ Redis для хранения номеров дел, превышающих лимит
Основные методы

handle()

Основной метод класса, который запускается при вызове команды через Artisan. Реализует полный цикл синхронизации дел для мониторинга.

Copypublic function handle()
{
    $this->logger->info("=== Начало синхронизации дел для мониторинга ===");

    // Если вызван флаг --clear, полностью чистим мониторинг + Redis
    if ($this->option('clear') && !$this->option('test')) {
        $this->clearAllMonitoredCases();
        return 0;
    }

    // Дальнейший код метода...
}
Алгоритм работы метода handle()
Получение валидных номеров дел из AmoCRM
Получение текущего списка дел на мониторинге через API
Удаление с мониторинга дел, которых нет в базе данных
Определение дел, которые уже мониторятся
Выделение дел, которых нет на мониторинге
Проверка дел со статусом "accepted" в Redis
Определение дел, для которых необходимо вызвать caseRetrieve
Вызов caseRetrieve для новых дел
Формирование итогового набора дел для мониторинга
Обработка "overflow" дел (превышающих лимит)
Проверка необходимости обновления мониторинга
Передача актуальных дел в контроллер для дальнейшего обновления
Вспомогательные методы

getValidCaseNumbersFromAmo()

Метод получает и валидирует номера дел из базы данных AmoLead. Если номер дела не соответствует формату, создается задача в amoCRM.

Copyprivate function getValidCaseNumbersFromAmo(): array
{
    $fieldId = config('amofields.case_number');

    $leads = AmoLead::where("status_id", config("amocrm.statuses.bfl.procedure_started.status_id"))
        ->get();

    $cases = [];
    foreach ($leads as $lead) {
        $json = json_decode($lead->data_json, true);
        $caseNumber = null;

        // Ищем нужное кастомное поле
        foreach ($json as $field_info) {
            if ($field_info["field_id"] == $fieldId) {
                $caseNumber = $field_info["values"][0]["value"] ?? null;
                break;
            }
        }

        // Проверяем корректность номера дела
        if (!$this->option('test') && !AmoHelper::validateCaseNumber($caseNumber, $lead->lead_id, $this->logger)) {
            continue;
        }

        if ($caseNumber) {
            $cases[] = $caseNumber;
        }
    }
    return array_values(array_unique($cases));
}
clearAllMonitoredCases()

Метод выполняет полную очистку всех дел с мониторинга и связанных ключей Redis. Вызывается при использовании флага --clear.

Copyprivate function clearAllMonitoredCases(): void
{
    $api = new CyberJusticeApiService();
    $monitorList = $api->getCaseMonitorList();

    if (is_array($monitorList)) {
        foreach ($monitorList as $item) {
            if (!empty($item['id'])) {
                sleep(1); // Учитываем rate-limit
                $api->deleteCaseMonitor($item['id']);
            }
        }
    }

    // Очищаем Redis
    Redis::del(self::RETRIEVE_REDIS_KEY);
    Redis::del(self::OVERFLOW_REDIS_KEY);

    $this->logger->info("Все дела удалены с мониторинга, Redis ключи очищены.");
}
Константы

Класс содержит несколько важных констант:

MONITORING_LIMIT - максимальное количество дел для мониторинга (500)
RETRIEVE_REDIS_KEY - ключ Redis для хранения номеров дел в статусе "accepted"
OVERFLOW_REDIS_KEY - ключ Redis для хранения номеров дел, превышающих лимит
Примеры использования

Пример 1: Стандартный запуск синхронизации

Copyphp artisan cyberjustice:sync:case-monitoring
Этот вызов запустит синхронизацию дел для мониторинга в штатном режиме. Будут выполнены все шаги алгоритма и произведена синхронизация с API CyberJustice.

Пример 2: Запуск в тестовом режиме

Copyphp artisan cyberjustice:sync:case-monitoring --test
Запуск в тестовом режиме выполнит все те же действия, что и стандартный запуск, но не будет производить реальных изменений в системе мониторинга. Это полезно для отладки и проверки алгоритма.

Пример 3: Очистка мониторинга

Copyphp artisan cyberjustice:sync:case-monitoring --clear
Этот вызов удалит все дела с мониторинга и очистит связанные ключи Redis.

Пример 4: Программное использование в другом классе

Copyuse App\Console\Commands\CyberJustice\SyncCaseMonitoringCommand;
use Illuminate\Support\Facades\Artisan;

// В методе какого-либо класса
public function refreshMonitoring()
{
    $exitCode = Artisan::call('cyberjustice:sync:case-monitoring');
    
    return $exitCode === 0 
        ? 'Синхронизация успешно выполнена' 
        : 'Ошибка при синхронизации';
}
Полный жизненный цикл

Для лучшего понимания работы команды, рассмотрим полный жизненный цикл процесса синхронизации:

Инициализация:
Создание экземпляра класса, настройка логгера
Проверка опций запуска (--clear, --test)
Получение данных:
Получение валидных номеров дел из AmoCRM
Получение текущего списка дел на мониторинге через API CyberJustice
Анализ различий:
Определение дел, которые надо удалить с мониторинга (нет в AmoCRM)
Определение дел, которые остаются на мониторинге
Определение новых дел, которые нужно добавить
Обработка новых дел:
Проверка статуса "accepted" в Redis
Вызов caseRetrieve для новых дел
Обработка ответов API (found, accepted)
Формирование итогового списка:
Объединение текущих и новых дел
Обработка overflow (превышение лимита)
Синхронизация с API:
Вызов syncCases с итоговым списком дел
Передача актуальных дел в контроллер для обновления
Возможные проблемы и их решения

Проблема 1: Превышение лимита дел

Проблема: Если количество дел для мониторинга превышает MONITORING_LIMIT (500), возникает overflow.

Решение: Класс автоматически обрабатывает эту ситуацию, помещая лишние дела в Redis под ключом OVERFLOW_REDIS_KEY. При освобождении места в мониторинге, эти дела будут добавлены при следующем запуске команды.

Copy// Обработка overflow (из метода handle)
if (count($finalCasesForMonitor) > self::MONITORING_LIMIT) {
    // Обрежем и «лишние» положим в overflow
    $overflow = array_slice($finalCasesForMonitor, self::MONITORING_LIMIT);
    foreach ($overflow as $num) {
        Redis::sadd(self::OVERFLOW_REDIS_KEY, $num);
    }
    $this->logger->warning("Часть дел ушла в overflow (превышен лимит): ", $overflow);

    // Оставляем только первые 500 в мониторинге
    $finalCasesForMonitor = array_slice($finalCasesForMonitor, 0, self::MONITORING_LIMIT);
}
Проблема 2: Rate-limit API

Проблема: API CyberJustice может иметь ограничения на количество запросов в определенный промежуток времени.

Решение: Между запросами к API добавлены задержки (sleep) для соблюдения rate-limit:

Copy// Пример из метода handle
foreach ($needRetrieve as $caseNumber) {
    sleep(1.5); // Учитываем rate-limit
    $retr = $api->caseRetrieve($caseNumber, false);
    // ...
}
Проблема 3: Некорректные номера дел

Проблема: В AmoCRM могут быть указаны некорректные номера дел.

Решение: Метод getValidCaseNumbersFromAmo() проверяет корректность номеров дел с помощью AmoHelper::validateCaseNumber(), который создает задачу в amoCRM для исправления, если номер некорректен.

Copy// Проверяем корректность номера дела
if (!$this->option('test') && !AmoHelper::validateCaseNumber($caseNumber, $lead->lead_id, $this->logger)) {
    continue;
}
Этот мануал предоставляет подробную информацию о классе SyncCaseMonitoringCommand, его методах, константах и примерах использования. Команда играет важную роль в синхронизации дел между базой данных AmoCRM и системой мониторинга CyberJustice.
