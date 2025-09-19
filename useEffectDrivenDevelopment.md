# useEffect‑driven development: что это, почему это антипаттерн и как делать правильно

## Что это
useEffect‑driven development — стиль написания React‑компонентов, в котором значимая часть логики (инициализация, вычисления, «синхронизация» состояний, загрузка данных, валидация) кладётся в `useEffect`, а рендер остаётся «тонким». Компонент превращается в «суп из эффектов»: цепочки `useEffect` с запутанными зависимостями, которые гоняют состояние туда‑сюда.

## Почему это антипаттерн
- Нарушение декларативности: бизнес‑логика прячется в побочных эффектах вместо того, чтобы быть выраженной через данные и рендер.
- Хрупкие зависимости: легко ошибиться с массивом deps → гонки, двойные/пропущенные обновления, фликание UI.
- Strict Mode в Dev вызывает эффекты дважды (React 18+) и выявляет неидемпотентные эффекты → дубли запросов.
- Лишние рендеры и «водопады» запросов: эффекты запускаются после коммита, добавляя промежуточные состояния и задержки.
- Сложнее тестировать и сопровождать: порядок срабатывания/очистки неочевиден, логика размазана.
- Дублирование состояния: «синхронизация» производных значений через `useEffect` вместо вычисления при рендере.

## Характерные симптомы
- «Синхронизирую» одно состояние с другим через `useEffect`.
- Делаю вычисления/валидацию в `useEffect` вместо вычисления при рендере.
- Любая загрузка данных — через `useEffect` на mount с ручными флагами отмены.
- Храню в состоянии то, что вычисляется из `props`/`state`.
- Бесконечные или фликкерящие циклы из‑за `setState` внутри эффекта с зависимостью на это же состояние.

---

## Антипаттерны и как правильно

### 1) Производное состояние
Плохо:
```tsx
const [count, setCount] = useState(0);
const [double, setDouble] = useState(0);
useEffect(() => setDouble(count * 2), [count]);
```

Хорошо:
```tsx
const [count, setCount] = useState(0);
const double = count * 2; // или useMemo, если вычисление тяжёлое
```

### 2) Инициализация состояния «после маунта»
Плохо:
```tsx
const [initial, setInitial] = useState<number>();
useEffect(() => {
  setInitial(computeExpensive(props.seed));
}, [props.seed]);
```

Хорошо (ленивый инициализатор):
```tsx
const [initial, setInitial] = useState(() => computeExpensive(props.seed));
```

### 3) «Синхронизация» props → state
Плохо:
```tsx
const [open, setOpen] = useState(props.open);
useEffect(() => setOpen(props.open), [props.open]);
```

Хорошо:
- Контролируемый компонент: используйте `props.open` напрямую.
- Или неконтролируемый: `const [open, setOpen] = useState(defaultOpen)` и больше не «синхронизируйте» с пропом.

### 4) Валидация/фильтрация через эффект
Плохо:
```tsx
const [errors, setErrors] = useState<string[]>([]);
useEffect(() => setErrors(validate(values)), [values]);
```

Хорошо:
```tsx
const errors = validate(values); // или useMemo, если дорого
// const errors = useMemo(() => validate(values), [values]);
```

### 5) Загрузка данных «на маунт» вручную
Плохо (гонки, дубль в Strict Mode):
```tsx
useEffect(() => {
  let aborted = false;
  fetch(`/api/user/${id}`)
    .then(r => r.json())
    .then(d => { if (!aborted) setUser(d); });
  return () => { aborted = true; };
}, [id]);
```

Лучше:
- Использовать библиотеку данных (TanStack Query/SWR), решающую кеш, гонки, повтор, инвалидизацию:
```tsx
const { data: user, isLoading } = useQuery({
  queryKey: ['user', id],
  queryFn: () => fetch(`/api/user/${id}`).then(r => r.json()),
});
```
- В Next.js/SSR — загружать на сервере (RSC, loaders, Suspense).
- Загружать по событию, если это пользовательское действие (а не «магия на маунт»).

### 6) Поддержание «синхронизированных» кусков состояния
Плохо:
```tsx
const [query, setQuery] = useState('');
const [filtered, setFiltered] = useState<Item[]>([]);
useEffect(() => {
  setFiltered(items.filter(i => i.name.includes(query)));
}, [items, query]);
```

Хорошо:
```tsx
const filtered = useMemo(
  () => items.filter(i => i.name.includes(query)),
  [items, query]
);
```

### 7) Бесконечная петля
Плохо:
```tsx
useEffect(() => {
  setCount(count + 1);
}, [count]); // бесконечный цикл
```

Хорошо:
- Изменяйте состояние в обработчиках событий или используйте `useReducer`:
```tsx
type Action = { type: 'inc' } | { type: 'dec' };
function reducer(state: number, action: Action) {
  switch (action.type) {
    case 'inc': return state + 1;
    case 'dec': return state - 1;
    default: return state;
  }
}
const [count, dispatch] = useReducer(reducer, 0);
```

---

## Когда `useEffect` уместен
- Синхронизация с внешними системами: подписки, веб‑сокеты, таймеры, интеграции с нестандартным DOM/SDK.
- Императивные эффекты после коммита: логирование, измерения, фокус, прокрутка.
- Синхронизация с внешним хранилищем/ресурсом (с корректной очисткой).
Подсказка: если значение можно вычислить во время рендера — это не для `useEffect`.

## Практический чеклист
- Это производное от props/state? Вычислите в рендере или через `useMemo`.
- Это пошаговая пользовательская логика? Перенесите в обработчики событий или `useReducer`.
- Это данные/кеш? Используйте TanStack Query/SWR или серверную загрузку (RSC/Suspense).
- Нужно связать компонент с «внешним миром» (подписка, таймер, DOM‑API)? Да, `useEffect` с аккуратным cleanup.
- Нужна «инициализация» из props? Используйте ленивый `useState` или сделайте компонент контролируемым.

## Полезные ссылки
- [You Might Not Need an Effect — React docs](https://react.dev/learn/you-might-not-need-an-effect)
- [Synchronizing with Effects — React docs](https://react.dev/learn/synchronizing-with-effects)
- [A Complete Guide to useEffect (Dan Abramov)](https://overreacted.io/a-complete-guide-to-useeffect/)
- [TanStack Query](https://tanstack.com/query/latest)
- [SWR](https://swr.vercel.app/)
- [Mobx Reactions](https://mobx-cookbook.github.io/beware-reactions)
- [Subscribers are notified in random order - Observer Pattern](https://refactoring.guru/design-patterns/observer)

Коротко: `useEffect` — инструмент для синхронизации с внешним миром, а не для организации бизнес‑логики и вычислений. Чем меньше «эффектов ради вычислений», тем предсказуемее и проще ваш код.
