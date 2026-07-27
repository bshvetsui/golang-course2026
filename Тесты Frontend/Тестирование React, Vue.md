# Тестирование React / Vue (2026)

| Вид             | Инструменты                        | Как выглядит                                | Что проверяет                                                                                          |
| --------------- | ---------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **E2E**         | Playwright                         | `page.goto() → click() → fill() → expect()` | Полный путь пользователя: логин, регистрация, покупка, CRUD, переходы между страницами, работа с API   |
| **Visual**      | Playwright, Chromatic, Percy       | `expect(page).toHaveScreenshot()`           | Что после изменений не поехала верстка: размеры, отступы, цвета, шрифты, адаптив, исчезнувшие элементы |
| **Integration** | React/Vue Testing Library + Vitest | `render() → user.click() → expect()`        | Взаимодействие компонентов: форма → запрос → обновление UI, модалки, роутинг, Context/Pinia/Redux      |
| **Unit**        | Vitest (Jest)                      | `expect(calc()).toBe(...)`                  | Чистая логика: utils, hooks/composables, валидация, форматирование, бизнес-правила                     |
| **Component**   | RTL/VTL                            | `render(<Table />)`                         | Только сложные компоненты: DataGrid, Chart, Drag&Drop, Editor, виртуализация                           |


### Примеры

**E2E (Playwright)**

```ts
await page.goto('/login')
await page.getByLabel('Email').fill('user@test.com')
await page.getByRole('button', { name: 'Login' }).click()
await expect(page).toHaveURL('/dashboard')
```

**Visual Regression**

```ts
await page.goto('/profile')
await expect(page).toHaveScreenshot('profile.png')
```

**Integration**

```ts
render(<LoginForm />)

await user.type(screen.getByLabelText('Email'), 'user@test.com')
await user.click(screen.getByRole('button'))

expect(await screen.findByText('Welcome')).toBeVisible()
```

**Unit**

```ts
expect(calculateDiscount(100, 20)).toBe(80)
expect(formatPrice(100)).toBe('$100')
```