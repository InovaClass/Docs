# Arquitetura do inovaclass-app

Essa documentação está em constante desenvolvimento e tem o intuito de apresentar a arquitetura do projeto e demonstrar a implementação de novas funcionalidades.

## Estrutura de pastas

Todo código que representa funcionalidade estará presente na pasta `src`, que pode conter as seguintes subpastas:

- `assets`: Contém arquivos estáticos, dentre eles, os arquivos **.svg**
- `components`: Contém os componentes reutilizáveis. Exemplo: `Button.tsx`, `Header.tsx`
- `contexts`: Contém os contextos globais. Exemplo: **ThemeContext**
- `data`: Contém dados padrão utilizados. Exemplo: pontos comportamentais
- `environment`: Configurações de serviços. Exemplo: **Firebase**
- `hooks`: Contém os hooks customizados. Exemplo: **UseThemeContext**
- `messages`: Contém as mensagens de sucesso e erro
- `models`: Contém a definição dos métodos de cada serviço
- `pages`: Contém as páginas do app. Exemplo: `HomePage.tsx`, `LoginPage.tsx`
- `providers`: Contém os provedores globais. Exemplo: **ThemeProvider**
- `services`: Contém as consultas dos serviços externos. Exemplo: consultas no banco de dados
- `themes`: Contém os temas do app (dark, light, etc.)
- `types`: Contém as definições dos tipos de dados referentes às `collections`
- `utils`: Contém as funções genéricas. Deve conter arquivos separados para funções específicas como:
  - `generics`: Funções que não se enquadram em uma nomenclatura específica. Exemplo: `delay`
  - `getters`: Funções com o prefixo `get`
  - `formatters`: Funções com o prefixo `format`
  - `storage`: Funções que manipulam o **local storage**
  - `validators`: Contém as funções de validação. Exemplo: `cpfIsValid`

À medida que o **app** for crescendo, surgirá a necessidade de criação de novas pastas. Devemos nos atentar à semântica e separação de arquivos.

Outro ponto interessante é criar um arquivo `index.ts` para exportar os arquivos internos. Exemplo:

```ts
    export * from './generics`
    export * from './getters`
```

Dessa forma, é possível importar essas funções através do _alias_ `@/utils`, `@/components`, etc.

## Criação de novos componentes e páginas

É possível criar um novo componente através do comando:

```sh
npm run generate MyComponent
```

Dessa forma, será criado o componente `MyComponent` dentro da pasta `components`, seguindo o modelo presente em `generators/templates`. É necessário exportar manualmente no arquivo `index.ts` conforme citado no tópico anterior. Exemplo:

```ts
export * from './MyComponent'
```

A pasta `src/components` deve conter apenas os componentes que são utilizados em mais de uma página, e as páginas devem estar na pasta `pages` e seguir a relação da `url`.

Ao criar a página `Teacher` renderizada através da _url_ `/teacher`, o componente principal estará contido no arquivo `index.tsx` e os componentes utilizados apenas nessa página estarão presentes na mesma pasta. No caso de mais de um componente, é recomendado criar uma nova pasta `components` nesse diretório.

Para a _url_ `/teacher/game`, a página `Game` estará na pasta `Teacher`. Atualmente, temos a seguinte estrutura:

```
src/pages/Teacher
├── Game
│   ├── components
│   │   ├── Actions.tsx
│   │   ├── BehaviorPoints.tsx
│   │   ├── ModifyPointsManually.tsx
│   │   ├── StudentsForm.tsx
│   │   ├── StudentsTable.tsx
│   │   ├── index.ts
│   │   ├── schema.ts
│   │   ├── styles.ts
│   │   ├── types.ts
│   │   └── utils.ts
│   ├── index.tsx
│   ├── use-game.ts
│   ├── use-load-data.ts
│   └── use-mutate-data.ts
├── components
│   ├── Header.tsx
│   ├── MyClassCard.tsx
│   ├── MyClassForm.tsx
│   ├── MyClasses.tsx
│   ├── Welcome.tsx
│   ├── index.ts
│   ├── schema.ts
│   └── styles.ts
├── index.tsx
└── use-my-classes.ts
```

## Componentes de UI e Hooks

Atualmente, o app conta com componentes e _hooks_ customizados para lidar com eventos na interface. Dentre os componentes, temos: `Alert`, `Dialog`, `Loader`, `Menu`, `Modal`, etc. E dentre os _hooks_, temos: `useAlert`, `useCheck`, `useDialog`, `useMenu`, `useOpen`, etc.

### Modais

Para criar um modal, devemos utilizar a seguinte estrutura:

```tsx
import { Modal } from '@/components'

export type TMyModalProps = {
  isOpened: boolean
  handleClose(): void
}

export const MyModal = ({ isOpened, handleClose }: TMyModalProps) => {
  return (
    <Modal title="Meu Modal" isOpened={isOpened} handleClose={handleClose}>
      conteúdo
    </Modal>
  )
}
```

Já no componente pai, temos:

```tsx
export const MyComponent = () => {
  const { open, handleOpen, handleClose } = useOpen()
  return (
    <>
      <Button onClick={handleOpen}>Abrir Modal</Button>
      <MyModal isOpened={open} handleClose={handleClose} />
    </>
  )
}
```

### Dialogs

Para _Dialogs_, temos uma estrutura parecida com a dos _Modais_, porém, nesse caso, iremos utilizar o _hook_ `useDialog` ao invés do genérico `useOpen`. Um exemplo de uso é quando o usuário for deletar algum dado. Para esses casos, devemos exibir um _Dialog_ exigindo a confirmação dessa ação. Exemplo:

```tsx
export const MyComponent = () => {
  const { dialogProps, handleDialogOpen, handleDialogClose } = useDialog()

  const handleDelete = () => {
    // lógica de exclusão
    handleDialogClose()
  }

  return (
    <>
      <Button
        onClick={() =>
          handleDialogOpen({
            title: 'Deseja deletar esse dado?',
            handleAccept: handleDelete
          })
        }
      >
        Excluir
      </Button>
      <Dialog {...dialogProps} />
    </>
  )
}
```

### Fique atento à pasta hooks

A pasta `src/hooks` contém funções extremamente importantes para a construção da aplicação, e já existe implementação de todos eles no código. Por isso, é importante ter o conhecimento de todos eles.

## Criação e implementação de requisições à API

Dada a _collection_ `myClasses`, devemos primariamente definir o `type`, o `model` e o `service`. Dessa forma, teremos a estrutura pronta para ser utilizada em nossos componentes.

Seguindo o exemplo, devemos realizar os seguintes passos para definir o `type`, o `model` e o `service`:

1. Criar o `type`

`types/my-class.ts`

```ts
export type TMyClassStatus = 'active' | 'archived'

export type TMyClass = {
  id: string
  userId: string
  status: TMyClassStatus
  name: string
  description: string | null
  createdAt: string
  updatedAt: string
}
```

`types/index.ts`

```ts
export * from './my-class'
```

2. Criar o `model`

`models/my-class.ts`

```ts
import { TMyClass, TMyClassStatus } from '@/types'

export type TMyClassModel = {
  createMyClass(myClass: Omit<TMyClass, 'id'>): Promise<void>
  readMyClass(id: string): Promise<TMyClass | null>
  readMyClasses(args: {
    userId: string
    status: TMyClassStatus
  }): Promise<TMyClass[]>
  updateMyClass(myClass: TMyClass): Promise<void>
}
```

3. Criar o `service`

`services/my-class.ts`

```ts
import {
  addDoc,
  collection,
  doc,
  getDoc,
  query,
  updateDoc,
  where
} from 'firebase/firestore'
import { db } from '@/environment'
import { TMyClassModel } from '@/models'
import { TMyClass } from '@/types'
import { getDocsData, getTimestamp } from '@/utils'

const collectionName = 'myClasses'

export const myClassService: TMyClassModel = {
  createMyClass: async (myClass) => {
    await addDoc(collection(db, collectionName), myClass)
  },
  readMyClass: async (id) => {
    const docRef = doc(db, collectionName, id)
    const docSnap = await getDoc(docRef)
    return docSnap.exists() ? (docSnap.data() as TMyClass) : null
  },
  readMyClasses: async ({ status, userId }) => {
    const q = query(
      collection(db, collectionName),
      where('userId', '==', userId),
      where('status', '==', status)
    )
    const data = await getDocsData(q)
    return data as TMyClass[]
  },
  updateMyClass: async (myClass) => {
    await updateDoc(doc(db, collectionName, myClass.id), {
      ...myClass,
      updatedAt: getTimestamp()
    })
  }
}
```

Com esses três passos realizados, agora é possível utilizar essas funções para fazer as requisições, que deverão ser feitas usando o `React Query`.

Seguindo o exemplo, após a criação do componente `MyClasses.tsx`, foi criado o _hook_ `use-my-classes.ts`, onde as requisições são instanciadas da seguinte forma:

```ts
const { createMyClass, readMyClasses, updateMyClass } = myClassService
```

Para os métodos do tipo `get`, utilizamos a função `useQuery`, onde devemos passar os parâmetros conforme o exemplo abaixo:

```ts
const {
  data: myClasses,
  error: readMyClassesError,
  isPending: isMyClassesPending,
  refetch
} = useQuery({
  queryKey: ['readMyClasses', args],
  queryFn: async () => await readMyClasses(args)
})
```

Para lidar com os alertas de erro ou sucesso, temos o _hook_ `useAlert`:

```ts
const { alert, handleAlertOpen } = useAlert()
```

Dessa forma, podemos monitorar as mudanças de estado da variável `readMyClassesError` através do `useEffect` e chamar o método `handleOpenAlert`:

```ts
useEffect(() => {
  if (!readMyClassesError) return
  console.error(readMyClassesError)
  handleAlertOpen({
    severity: 'error',
    title: myClassError.readAll
  })
}, [readMyClassesError])
```

Para as mutações, devemos seguir a seguinte estrutura:

```ts
const { mutate: createMyClassMutate, isPending: isCreateMyClassPending } =
  useMutation({
    mutationKey: ['createMyClass'],
    mutationFn: createMyClass,
    onSuccess: () => {
      refetch()
      handleAlertOpen({
        severity: 'success',
        title: myClassSuccess.create
      })
      handleFormClose()
    },
    onError: () =>
      handleAlertOpen({
        severity: 'error',
        title: myClassError.create
      })
  })
```

Com isso, temos o que é necessário pa lidar com as requisições do nosso aplicativo
