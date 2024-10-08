# Рефакторинг кода и замечания по его улучшению

В этом репозитории представлены различные примеры кода, используемого в Unity проектах. Ниже приведены наблюдения и предложения по улучшению для каждого из предоставленных фрагментов кода.

## 1. Манипуляция здоровьем игрока

### Первый Вариант

В первом варианте создается объект игрока, и его здоровье изменяется с помощью метода "ударить" игрока. Однако есть несколько проблем:
- **Нарушение инкапсуляции:** Метод `SetHealth` позволяет внешнему коду манипулировать здоровьем игрока, что нарушает принцип инкапсуляции. Логика расчета урона должна находиться внутри класса `Player`, как это сделано во втором варианте с методом `Hit`.
- **Опечатка:** В коде присутствует опечатка — `NewPayerHealth` вместо `NewPlayerHealth`.
- **Неочевидная логика:** Для нанесения урона нужно сначала рассчитать `player.Health - Damage`, что делает код менее чистым. Эта логика должна быть внутри класса `Player`.

### Второй Вариант

Второй вариант мне нравится больше, так как он использует конфигурации и подгружает настройки извне:
- **Конфигурации:** Мне нравится концепция конфигов, но есть несколько замечаний:
  - На этапе разработки может понадобиться создание объектов `Player` и `Settings` вручную. У `Player` свойство `Health` имеет приватный сеттер и не инициализируется в конструкторе, что может стать проблемой. Возможно, стоит добавить дополнительный конструктор для ручной инициализации.
  - В классе `Settings` также не инициализируется поле `Damage`, и у него нет сеттера, что приведет к тому, что `JsonUtility` не сможет его установить.
  - Подгрузку конфигов лучше вынести в отдельный метод, хотя для тестового примера допустимо оставить это в методе `Main`.

В реальном проекте я бы выбрал второй вариант, модифицировав и доработав его.

## 2. Отслеживание урона и манипуляция виджетом здоровья

Оба варианта отслеживают нанесенный игроку урон и управляют виджетом для отображения здоровья, изменяя цвет текста с текущим уровнем здоровья.
- **Непонятная логика:** В текущей логике текст меняет цвет на красный, если урон больше 10, и на белый, если меньше. На мой взгляд, это может выглядеть странно. Было бы понятнее, если бы текст менял цвет на красный при низком уровне здоровья (например, < 30).
- **Magic Numbers:** Условие `newHealth - oldHealth < -10` не интуитивно понятно. Лучше вычитать меньшее из большего и убрать отрицательное значение. Число 10 лучше вынести в константу или `readonly` переменную.
- **Нарушение инкапсуляции:** Прямое изменение параметров виджета (`healthView.Color`, `healthView.Text`) нарушает инкапсуляцию. Лучше использовать методы вроде `ChangeColor(Color newColor)` и `ChangeText(string newText)`.

В первом варианте используются делегаты для передачи старого и нового значения здоровья, во втором — только уведомление о получении урона, а старое значение сохраняется локально. Лично мне кажется, что передача нового значения здоровья излишняя, так как его можно получить через `player.Health`, но передача значения урона (`DamageAmount`) может быть полезной.

Также в первом варианте метод подписки на событие сразу вызывает метод, что может быть неочевидным решением. Лучше избегать вызова метода при подписке и вынести инициализацию UI в отдельный метод.

## 3. Поиск пути

Оба варианта кода реализуют функциональность поиска пути для игрока при наличии врага. Наибольшая проблема, на мой взгляд - то что сейчас поиск пути выполняется каждый кадр, это может быть дорого по ресурсам. Переменная `List<Vector2> activeWalkPath` говорит о том, что у нас меняются только X и Y координаты, соответственно на локации есть некая сетка. 
- **Оптимизация:** Выполнять поиск пути каждый кадр может быть слишком ресурсоемким. Поскольку пути зависят только от координат, их можно заранее посчитать и закешировать, что снизит нагрузку. В зависимости от игры может понадобиться пересчитывать путь в определенных случаях, но это все равно будет происходить реже, чем каждый кадр.
- **Приватные методы:** Методы Unity, такие как `Update`, лучше оставлять приватными и не вызывать их из других классов.
- **Сигнатура метода:** Во втором примере метод `private List<Vector2> TryBuildPathToCoord(Vector2 target)` можно улучшить, заменив его на `private bool TryBuildPathToCoord(Vector2 target, out List<Vector2> coords)`.

## 4. Реализация системы читов

Оба подхода реализуют функциональность системы читов в Unity.
- **Первая реализация:** Этот вариант выглядит более масштабируемым и инкапсулированным. Легко добавлять новые читы через интерфейс `ICheatProvider`.
- **Вторая реализация:** Более простая и легковесная, подходит для небольших проектов. В больших проектах этот вариант может превратиться в громоздкую и неудобную систему.

В обоих вариантах инстанцирование объектов лучше вынести из кода, и использовать строки типа `"Cheat health"` как константы. Скажу честно, мое мнение про чит-функции — обычно, я смотрю реализацию между строк и закрываю глаза на многие огрехи, так как для читов в первую очередь важна функциональность, и микрооптимизация здесь будет совсем лишней, ведь на стадии продакшена читы не должны попадать в билд и не будут никаким образом влиять на производительность. Важно убедиться, что читы не работают в продакшен-билде, и никакие подсистемы читов не инициализируются. Для этого можно использовать `#define` или другие механизмы компиляции.

## 5. Рефакторинг кода в CleanupTest

В рефакторинге кода CleanupTest были внесены следующие изменения для улучшения читаемости и поддерживаемости:
- **Объединение условий:** Повторяющиеся проверки были объединены с использованием оператора `||`, что сделало код более компактным.
- **Использование интерфейсов:** Вместо типа `object` использованы интерфейсы `ITargetable` и `ITargetableContainer`, что улучшает типобезопасность и читабельность.
- **Структурирование методов:** Код разделен на методы с чётко определенными задачами, что упрощает его понимание и поддержку.

```csharp
namespace Cleanup
{
    internal class Program
    {
        private const double TargetChangeTime = 1;

        private double _previousTargetSetTime;
        private bool _isTargetSet;
        private ITargetable _lockedCandidateTarget;
        private ITargetable _lockedTarget;
        private ITargetable _target;
        private ITargetable _previousTarget;
        private ITargetable _activeTarget;
        private ITargetableContainer _targetInRangeContainer;

        public void CleanupTest(Frame frame)
        {
            try
            {
                ValidateLockedTargets();

                _isTargetSet = TrySetActiveTargetFromQuantum(frame) || 
                               TrySetTarget(_target, TargetChangeTime) || 
                               TrySetTarget(_lockedTarget) || 
                               TrySetTarget(_activeTarget) || 
                               TrySetTargetFromRange();

            }
            finally
            {
                FinalizeTargetSelection();
            }
        }

        private void ValidateLockedTargets()
        {
            ValidateAsTarget(ref _lockedCandidateTarget);
            ValidateAsTarget(ref _lockedTarget);
        }

        private void ValidateAsTarget(ref ITargetable target)
        {
            if (target is { CanBeTarget: false })
            {
                target = null;
            }
        }

        private bool TrySetActiveTargetFromQuantum(Frame frame)
        {
            // Логика установки активного target предположительно из PhotonQuantum
            return false;
        }

        private bool TrySetTarget(ITargetable target, double timeConstraint = double.MaxValue)
        {
            if (target is { CanBeTarget: true } && (Time.time - _previousTargetSetTime < timeConstraint))
            {
                _target = target;
                return true;
            }

            return false;
        }

        private bool TrySetTargetFromRange()
        {
            _target = _targetInRangeContainer?.GetTarget();
            return _target != null;
        }

        private void FinalizeTargetSelection()
        {
            switch (_isTargetSet)
            {
                case true when _previousTarget != _target:
                    _previousTargetSetTime = Time.time;
                    break;
                case false:
                    _target = null;
                    break;
            }

            TargetableEntity.Selected = _target;
        }
    }

    public interface ITargetable
    {
        bool CanBeTarget { get; }
    }

    public interface ITargetableContainer
    {
        ITargetable GetTarget();
    }
}
```
