# Documentação do InovaClass

Esta documentação contém as instruções de como desenvolver o código do [inovaclass-app](https://github.com/InovaClass/inovaclass-app)

## Estrutura de pastas

Todo código que representa funcionalidade estará presente na pasta `src`, que pode conter as seguintes subpastas:

- `assets`: Contém arquivos estáticos como as imagens.
- `components`: Contém os componentes reutilizáveis. Exemplo: `Button.tsx`, `Header.tsx`
- `config`: Configurações de serviços. Exemplo: **Firebase**
- `contexts`: Contém os contextos globais. Exemplo: **ThemeContext**
- `data`: Contém dados padrão utilizados. Exemplo: **Pontos comportamentais**
- `hooks`: Contém os hooks customizados. Exemplo: **useMenu**
- `messages`: Contém as mensagens de sucesso e erro
- `models`: Contém a definição do `schema` de um dado a ser salvo no banco de dados
- `pages`: Contém as páginas do app. Exemplo: `Home.tsx`, `Login.tsx`
- `services`: Contém as consultas dos serviços externos. Exemplo: consultas no banco de dados
- `themes`: Contém os temas do app (dark, light, etc.)
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

Dessa forma, será criado o componente `MyComponent` dentro da pasta `components`, seguindo o modelo presente em `generators/templates`.

A pasta `src/components` deve conter apenas os componentes que são utilizados em mais de um local, e as páginas devem estar na pasta `pages` e seguir a relação da `url`.

Ao criar a página `Teacher` renderizada através da _url_ `/teacher`, o componente principal estará contido no arquivo `index.tsx` e os componentes utilizados apenas nessa página estarão presentes na mesma pasta. No caso de mais de um componente, é recomendado criar uma nova pasta `components` nesse diretório.

Para a _url_ `/teacher/game`, a página `Game` estará na pasta `Teacher` e assim por diante. Na estrutura abaixo, a página `Objective` estará presente na _url_ `/teacher/game/missions/mission/objective` e a página `Profile` estará presente na _url_ `/teacher/profile`

```
src/pages/Teacher
├── Game
│   ├── Missions
│   │   ├── Mission
│   │   │   ├── Objective
├── Profile
│   ├── index.tsx
│   └── style.ts
├── index.tsx
├── use-component-handler.ts
├── use-data-fetch.ts
└── use-data-mutation.ts
```

## Componentes de UI, Hooks e Contextos

Atualmente, o app conta com componentes, contextos e hooks customizados para lidar com eventos na interface. Dentre os contextos, temos: `Notification` e `Dialog`. Hooks para lidar com ações referentes ao `Menu`, `Stepper`, e `CheckList`, além de `Loader`, `Modal`.

### Modais

Para criar um modal, devemos utilizar a seguinte estrutura:

```tsx
import { Modal } from '@/components'

type TMyModalProps = {
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
  const [open, setOpen] = useState(false)
  return (
    <>
      <Button onClick={() => setOpen(true)}>Abrir Modal</Button>
      <MyModal isOpened={open} handleClose={() => setOpen(false)} />
    </>
  )
}
```

### Notificações e Confirmação de Ações

Para esses componentes, temos um contexto global dedicado. Para exibir uma notificação, basta instanciar o contexto e chamar a função `notification`, como pode ser visto abaixo:

```tsx
export const MyComponent = () => {
  const notification = useNotificationContext()

  const { mutate, isPending } = useMutation({
    mutationKey: ['create-item'],
    mutationFn: createItem,
    onSuccess: () => {
      notification({
        severity: 'success',
        title: 'Item criado com sucesso!'
      })
      refetch()
    },
    onError: (error) => {
      console.error(error)
      notification({
        severity: 'error',
        title: 'Não foi possível criar o item',
        description: 'Tente novamente',
        autoHideDuration: 5000
      })
    }
  })
  return <Button onClick={() => mutate(item)}>Criar item</Button>
}
```

Para exibir um `Dialog` para confirmar uma ação, podemos fazer da seguinte forma:

```tsx
export const MyComponent = () => {
  const dialog = useDialogContext()

  const handleDelete = () => {
    // lógica de exclusão
    dialog.close()
  }

  return (
    <Button
      onClick={() =>
        dialog.show({
          title: 'Deseja deletar esse dado?',
          handleAccept: handleDelete
        })
      }
    >
      Excluir
    </Button>
  )
}
```

### Fique atento à pasta hooks

A pasta `src/hooks` contém funções extremamente importantes para a construção da aplicação. Por isso, é importante ter o conhecimento de todos eles, dessa forma evitamos a duplicação de código e garantimos o melhor funcionamento da nossa aplicação.

## Criação dos serviços para requisições à API

Dada a _collection_ `classes`, devemos primariamente definir o `model` e o `service`. Dessa forma, teremos a estrutura pronta para ser utilizada em nossos componentes.

Seguindo o exemplo, devemos realizar os seguintes passos para criar os métodos relacionados a collection `classes`:

1. Criar o `model`

`models/class.ts`

```ts
export type TClassStatus = 'active' | 'archived'

export type TClassModel = {
  id: string
  userId: string
  status: TClassStatus
  name: string
  description: string | null
  createdAt: string
  updatedAt: string
}
```

```ts
export * from './class'
```

O `model` representa a estrutura do objeto `Class` no banco de dados.

---

2. Criar o `service`

`services/class.ts`

```ts
import {
  addDoc,
  collection,
  doc,
  query,
  updateDoc,
  where
} from 'firebase/firestore'
import { db } from '@/config'
import { TClassModel } from '@/models'
import { findDoc, getDocsData, getTimestamp } from '@/utils'

export class ClassService {
  private collectionName = 'classes'

  create = async (Class: Omit<TClassModel, 'id'>) => {
    await addDoc(collection(db, this.collectionName), Class)
  }

  findOne = async (id: string) => {
    const result = await findDoc<TClassModel>({
      id,
      collectionName: this.collectionName
    })
    return result
  }

  find = async ({ status, userId }: { status: string; userId: string }) => {
    const q = query(
      collection(db, this.collectionName),
      where('userId', '==', userId),
      where('status', '==', status)
    )
    const data = await getDocsData(q)
    return data as TClassModel[]
  }

  update = async (Class: TClassModel) => {
    await updateDoc(doc(db, this.collectionName, Class.id), {
      ...Class,
      updatedAt: getTimestamp()
    })
  }
}
```

Os `services` devem conter um ou mais métodos com a seguinte nomenclatura: `find`, `findOne`, `create`, `createMany`, `update`, `updateMany`, `remove` e `removeMany`.

Os métodos devem obrigatoriamente estarem escritos em forma e `arrow functions` para garantir a referência ao `this`, ou seja, dessa forma o `this.collectionName` retorna o valor corretamente.

---

Com esses passos realizados, agora é possível utilizar essas funções para fazer as requisições, que deverão ser feitas usando o `React Query`.

## Mensagens de Notificações

Deve ser criado um objeto contendo as mensagens para cada método de uma operação, como pode ser visto na definição do tipo abaixo:

```ts
type TMessageVariant = {
  find: string
  findMany: string
  create: string
  createMany: string
  update: string
  updateMany: string
  delete: string
  deleteMany: string
}

export type TMessage = {
  error: TMessageVariant
  success: TMessageVariant
}
```

Esses objetos, devem estar presentes na pasta `messages`. O exemplo abaixo define as mensagens das operações relacionadas às classes.

`messages/class.ts`

```ts
import { TMessage } from './types'

export const classMessage: TMessage = {
  error: {
    find: 'Não foi possível carregar a classe',
    findMany: 'Não foi possível carregar as classes',
    create: 'Não foi possível criar a classe',
    createMany: 'Não foi possível criar as classes',
    update: 'Não foi possível atualizar a classe',
    updateMany: 'Não foi possível atualizar as classes',
    delete: 'Não foi possível deletar a classe',
    deleteMany: 'Não foi possível deletar as classes'
  },
  success: {
    find: 'Classe carregada com sucesso',
    findMany: 'Classes carregadas com sucesso',
    create: 'Classe criada com sucesso',
    createMany: 'Classes criadas com sucesso',
    update: 'Classe atualizada com sucesso',
    updateMany: 'Classes atualizadas com sucesso',
    delete: 'Classe deletada com sucesso',
    deleteMany: 'Classes deletadas com sucesso'
  }
}
```

## Separação de arquivos na implementação de um componente

A medida que é a quantidade de linhas de código em um arquivo aumenta, é interessantes separar as funcionalidades em arquivos diferentes. Dessa forma, é possível compreender melhor a repensabilidade de cada funcionalidade.

Dependendo do componente, ele pode conter os seguintes arquivos:

- `index.tsx`: Arquivo principal
- `schema.ts`: Definição do objeto do formulário
- `style.ts`: Estilos CSS
- `types.ts`: Definição de tipos
- `use-component-handler.ts`: Hook com as funções de manipulação de estado
- `use-data-fetch.ts`: Hook para as funções de fetch de dados (GET) utilizando o `useQuery`
- `use-data-mutation.ts`: Hook para as funções de mutação de dados (POST, UPDATE, DELETE) utilizando o `useMutation`
- `utils.ts`: Funções genéricas a serem utilizadas apenas neste componentes

As vezes não é necessário criar um novo arquivo como o `types.ts`, `style.ts` e `utils.ts`. Mas se o componente necessitar de um esquema de formulário, fetch e mutação de dados, essas funções devem estar separadas conforme a descrição acima.

Também existem páginas que contem outros componentes além do `index.ts`. Exemplo: `Form.tsx`, `List.tsx`, `Card.tsx`. Cado quantidade deja maior que dois, é indicado criar uma nova pasta `componentes` dentro desse diretório.

### Exemplo prático

Dado o componente da página `Classes.tsx`, temos o arquivo `index.tsx`:

```tsx
import { useComponentHandler } from './use-component-handler'

export const TeacherGame = () => {
  const props = useComponentHandler()

  if (props.isLoading) return <Loader />

  return <Classes {...props} />
}
```

`use-data-fetch`:

```tsx
import { useEffect } from 'react'
import { useQueries } from '@tanstack/react-query'
import { ClassService, StudentService } from '@/services'
import { useNotificationContext } from '@/contexts'
import { genericMessage } from '@/messages'

const classService = new ClassService()
const studentService = new StudentService()

export const useDataFetch = ({
  userId,
  classId
}: {
  userId: string
  classId: string
}) => {
  const notification = useNotificationContext()

  const [
    { data: classData, error: classError, isFetching: isClassFetching },
    {
      data: studentsData,
      error: studentsError,
      isFetching: isStudentsFetching,
      refetch: studentsRefetch
    }
  ] = useQueries({
    queries: [
      {
        queryKey: ['findClass', classId],
        queryFn: async () => await classService.findOne(classId),
        enabled: !!classId
      },
      {
        queryKey: ['findStudents', classId],
        queryFn: async () => await studentService.find(classId),
        enabled: !!classId
      }
    ]
  })

  useEffect(() => {
    if (classError || studentsError) {
      classError && console.error(classError)
      studentsError && console.error(studentsError)
      notification({
        severity: 'error',
        title: genericMessage.error.findMany
      })
    }
  }, [classError, studentsError, notification])

  const isFetching = isClassFetching || isStudentsFetching

  return {
    classData,
    studentsData,
    isFetching,
    studentsRefetch
  }
}
```

Para os métodos do tipo `get`, utilizamos a função `useQuery` ou `useQueries`.

```ts
const {
  data: classes,
  error: classesError,
  isPending: isClassesPending,
  refetch
} = useQuery({
  queryKey: ['findClasses', args],
  queryFn: async () => await classService.find(args)
})
```

Para lidar com os alertas de erro ou sucesso, temos o _hook_ `useNotificationContext`:

```ts
const notification = useNotificationContext()
```

Dessa forma, podemos monitorar as mudanças de estado da variável `classesError` através do `useEffect` e chamar a função `notification`.

Para as mutações, devemos seguir a seguinte estrutura no arquivo `use-data-mutation`:

```ts
const { mutate: classMutationCreate, isPending: isClassMutationCreatePending } =
  useMutation({
    mutationKey: ['createClass'],
    mutationFn: ClassService.create,
    onSuccess: () => {
      refetch()
      notification({
        severity: 'success',
        title: classMessage.success.create
      })
      handleFormClose()
    },
    onError: (error) => {
      console.error(error)
      notification({
        severity: 'error',
        title: classMessage.error.create
      })
    }
  })
```

O arquivo `use-component-handler` une os dois hooks acima.

## Formulários

Os fomulários geralmente são criados utilizando um `schema` definido com a biblioteca `yup` e implementação com o `react-hook-form`.

O `schema` é definido conforme o código abaixo:

```ts
import * as yup from 'yup'

export const schema = yup.object({
  name: yup.string().required('Nome da classe é obrigatório'),
  description: yup.string()
})

export type TFormData = yup.InferType<typeof schema>
```

E a implementação segue o principio abaixo:

```tsx
import { useEffect, useMemo } from 'react'
import { useForm } from 'react-hook-form'
import { yupResolver } from '@hookform/resolvers/yup'
import { Button, FormControl, TextField } from '@mui/material'
import { Modal } from '@/components'
import { TClassModel } from '@/models'
import { schema, TFormData } from './schema'

export type TClassFormProps = {
  isOpened: boolean
  myClass: TClassModel | null
  handleClose(): void
  onSubmit(data: TFormData): void
}

export const ClassForm = ({
  isOpened,
  myClass,
  onSubmit,
  handleClose
}: TClassFormProps) => {
  const defaultValues: TFormData = useMemo(
    () => ({
      name: myClass?.name ?? '',
      description: myClass?.description ?? ''
    }),
    [myClass]
  )
  const {
    register,
    handleSubmit,
    reset,
    formState: { errors }
  } = useForm({
    resolver: yupResolver(schema),
    defaultValues
  })

  useEffect(() => {
    reset(defaultValues)
  }, [defaultValues, reset])

  return (
    <Modal
      title={!myClass ? 'Criar uma nova classe' : 'Editar classe'}
      isOpened={isOpened}
      handleClose={handleClose}
    >
      <FormControl
        component="form"
        role="form"
        noValidate
        onSubmit={handleSubmit(onSubmit)}
        fullWidth
      >
        <TextField
          {...register('name')}
          type="text"
          label="Nome da classe"
          defaultValue={myClass?.name}
          error={!!errors.name}
          helperText={errors.name?.message}
          sx={{ mb: 2 }}
        />
        <TextField
          {...register('description')}
          type="text"
          label="Descrição da classe"
          defaultValue={myClass?.description}
          error={!!errors.description}
          helperText={errors.description?.message}
          sx={{ mb: 2 }}
        />
        <Button type="submit" variant="contained">
          Salvar
        </Button>
      </FormControl>
    </Modal>
  )
}
```

Como o formulário acima apresenta estados para criação e edição de conteúdo, é necessário definir o `defaultValues` e o `reset` no `useEffect`.

E obrigatoriamente deve possui os parâmetros `error` e `helperText` para exibir as mensagens de validão definidas do `schema`.

Já a função `onSubmit` passada via props, é implementada no componente principal dentro do hook `use-component-handler`.

```tsx
const props = useComponentHandler()
return <ClassForm {...props} />
```

```ts
const { classMutationCreate, classMutationUpdate, isPending } = useDataMutation(
  {
    classId: myClass?.id,
    handleFormClose,
    refetch: classesRefetch
  }
)

const handleClassSubmit = async ({
  name,
  description
}: {
  name: string
  description?: string
}) => {
  const timestamp = getTimestamp()
  !myClass
    ? classMutationCreate({
        createdAt: timestamp,
        description: description || null,
        name,
        status: 'active',
        updatedAt: timestamp,
        userId
      })
    : classMutationUpdate({
        ...myClass,
        description: description || null,
        name,
        updatedAt: timestamp
      })
}
```
