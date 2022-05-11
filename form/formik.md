# formik

版本[2.2.9](https://github.com/jaredpalmer/formik/tree/formik%402.2.9)

用来处理 react 表单逻辑。

## demo

```jsx
import React from "react";
import { useFormik } from "formik";

const validate = (values) => {
  const errors = {};

  if (!values.firstName) {
    errors.firstName = "Required";
  } else if (values.firstName.length > 15) {
    errors.firstName = "Must be 15 characters or less";
  }
  // ...
  return errors;
};

const SignupForm = () => {
  const formik = useFormik({
    initialValues: {
      firstName: "",
      // ...
    },
    validate,
    onSubmit: (values) => {
      alert(JSON.stringify(values, null, 2));
    },
  });
  return (
    <form onSubmit={formik.handleSubmit}>
      <label htmlFor="firstName">First Name</label>
      <input
        id="firstName"
        name="firstName"
        type="text"
        onChange={formik.handleChange}
        onBlur={formik.handleBlur}
        value={formik.values.firstName}
      />
      {formik.errors.firstName ? <div>{formik.errors.firstName}</div> : null}
      // ...
      <button type="submit">Submit</button>
    </form>
  );
};
```

## How it works

首先从 demo 中可以看出整个 form 的状态是保存在 formik 中的。这个内部状态的类型是 FormikState。

```tsx
export interface FormikState<Values> {
  /** Form values */
  values: Values;
  /** map of field names to specific error for that field */
  errors: FormikErrors<Values>;
  /** map of field names to whether the field has been touched */
  touched: FormikTouched<Values>;
  /** whether the form is currently submitting */
  isSubmitting: boolean;
  /** whether the form is currently validating (prior to submission) */
  isValidating: boolean;
  /** Top level status state, in case you need it */
  status?: any;
  /** Number of times user tried to submit the form */
  submitCount: number;
}
```

values 保存着 form 中所有控件的 value，结构类似于下面这样，key 是控件的`name`。

```ts
export interface FormikValues {
  [field: string]: any;
}
```

每当控件的 value 有变化的时候会通过 formik 提供的`handleChange`函数，将 value 同步到 formik 中。除此之外，当我们使用非原生的 html 控件的时候，可以使用`setFieldValue`函数来同步 value。但需要显式地提供控件的`name`值。

```jsx
<input
  id="firstName"
  name="firstName"
  type="text"
  onChange={formik.handleChange}
  value={formik.values.firstName}
/>
// or
<CustomInput onChange={(value)=>{
	formik.setFieldValue("firstname", value)
}}
value={formik.values.firstname}
/>
```

在 handleChange 内部也是调用的 setFieldValue。只不过它是通过 change event 中的 target 来自动获取 name 值。

errors 表示表单中可能出现的错误。它的类型看起来比较复杂。实际上是利用了递归来表示自己的类型结构和上面的 FormikValues 是一样的。

```ts
/**
 * An object containing error messages whose keys correspond to FormikValues.
 * Should always be an object of strings, but any is allowed to support i18n libraries.
 */
export type FormikErrors<Values> = {
  [K in keyof Values]?: Values[K] extends any[]
    ? Values[K][number] extends object // [number] is the special sauce to get the type of array's element. More here https://github.com/Microsoft/TypeScript/pull/21316
      ? FormikErrors<Values[K][number]>[] | string | string[]
      : string | string[]
    : Values[K] extends object
    ? FormikErrors<Values[K]>
    : string;
};
```

formik 中的 validate 有三种，一种是单个控件的 validation，另外两种是全局的。

单个控件的 validate 函数会通过`registerField`注册到 formik 中。

```js
const fieldRegistry = React.useRef < FieldRegistry > {};
// ...
const registerField = React.useCallback((name: string, { validate }: any) => {
  fieldRegistry.current[name] = {
    validate,
  };
}, []);

const unregisterField = React.useCallback((name: string) => {
  delete fieldRegistry.current[name];
}, []);
```

然后执行 validation 操作的时候，会同时执行这三种，最后将所有的错误合并。注意这里使用了 Promise.all，这是为了兼容异步的 validation。

```ts
// Run all validations and return the result
const runAllValidations = React.useCallback(
  (values: Values) => {
    return Promise.all([
      runFieldLevelValidations(values),
      props.validationSchema ? runValidationSchema(values) : {},
      props.validate ? runValidateHandler(values) : {},
    ]).then(([fieldErrors, schemaErrors, validateErrors]) => {
      const combinedErrors = deepmerge.all<FormikErrors<Values>>(
        [fieldErrors, schemaErrors, validateErrors],
        { arrayMerge }
      );
      return combinedErrors;
    });
  },
  [
    props.validate,
    props.validationSchema,
    runFieldLevelValidations,
    runValidateHandler,
    runValidationSchema,
  ]
);
```

执行 validation 的时机有很多，例如在 submit 的时候、表单首次加载的时候、表单项失去焦点的时候(如果设置了`validateOnBlur`属性)、表单项 value 有变化的时候(如果设置了`validateOnChange`属性)等等。

touched 代表表单中的控件是否被动过，实际上是是否有触发过 blur 事件。它的结构也是和 FormikValues 一样的，一般用来控制 error message 何时显示。

整体上 formik 使用了`useReducer`和`Context`来管理分发状态。

`useFormik`hook 内部利用 useReducer hook 来保存整个表单的状态，实现并返回了许多函数。这些函数用来处理表单值的更新、验证以及表单提交等逻辑。

```ts
export function useFormik<Values extends FormikValues = FormikValues>(
  {
    // props about formik config
  }
) {
  // ...
  const [state, dispatch] = React.useReducer<
    React.Reducer<FormikState<Values>, FormikMessage<Values>>
  >(formikReducer, {
    values: props.initialValues,
    errors: props.initialErrors || emptyErrors,
    touched: props.initialTouched || emptyTouched,
    status: props.initialStatus,
    isSubmitting: false,
    isValidating: false,
    submitCount: 0,
  });
  // ...
  const ctx = {
    ...state,
    // ...
    handleBlur,
    handleChange,
    handleSubmit,
    setErrors,
    setFieldTouched,
    setFieldValue,
    setFieldError,
    setSubmitting,
    // ...
  };

  return ctx;
}
```

state 和函数需要通过 React 的 Context 机制来传递，这些在 Formik 组件中有体现。

```tsx
export function Formik<
  Values extends FormikValues = FormikValues,
  ExtraProps = {}
>(props: FormikConfig<Values> & ExtraProps) {
  const formikbag = useFormik<Values>(props);
  // ...
  if (__DEV__) {
    // ...
  }
  return (
    <FormikProvider value={formikbag}>
      {/*component or children*/}
    </FormikProvider>
  );
}
```

Formik 组件并不是必须的，我们也可以只使用 useFormik。但是与之配套的 Form 组件、Field 组件、FastField 组件和 ErrorMessage 组件等都需要 Context 来注入 state 和帮助函数。与其自己创建一个 Context，不如直接使用 Formik 组件来得方便。

在介绍剩下的这些组件之前，先看下 connect 这个高阶组件。实际上它的实现相当简单，只是通过 Context 来获取到 useFormik 执行后的返回值，再将这个值传递给需要的组件。

```tsx
export function connect<OuterProps, Values = {}>(
  Comp: React.ComponentType<OuterProps & { formik: FormikContextType<Values> }>
) {
  const C: React.FC<OuterProps> = (props: OuterProps) => (
    <FormikConsumer>
      {(formik) => {
        // ...
        return <Comp {...props} formik={formik} />;
      }}
    </FormikConsumer>
  );
  // ...
  return hoistNonReactStatics(
    C,
    Comp as React.ComponentClass<
      OuterProps & { formik: FormikContextType<Values> }
    > // cast type to ComponentClass (even if SFC)
  ) as React.ComponentType<OuterProps>;
}
```

在 ErrorMessage 组件中就是通过 connect 来得到某个控件的 touched 和 errors 信息。

Field 组件会为控件自动绑定上 formik 的一些监听函数和属性。用法有很多。Field 组件在内部会通过调用 useFormikContext 获取到一些方法和状态来实现相关表单控件逻辑。

```tsx
<Form>
  <Field type="email" name="email" placeholder="Email" />
  <Field as="select" name="color">
    <option value="red">Red</option>
    <option value="green">Green</option>
    <option value="blue">Blue</option>
  </Field>

  <Field name="lastName">
    {({
      field, // { name, value, onChange, onBlur }
      form: { touched, errors }, // also values, setXXXX, handleXXXX, dirty, isValid, status, etc.
      meta,
    }) => (
      <div>
        <input type="text" placeholder="Email" {...field} />
        {meta.touched && meta.error && (
          <div className="error">{meta.error}</div>
        )}
      </div>
    )}
  </Field>
  <Field name="lastName" placeholder="Doe" component={MyInput} />
  <button type="submit">Submit</button>
</Form>
```

除此之外还有一个 useField hook，内部也是调用了 useFormikContext，并返回了一组和控件逻辑相关的对象，包含了控件的状态和帮助函数。

上面提到 formik 使用了 Context 来传递 State，这就隐藏了一个性能问题。因为 State 是一个经常变化的数据，每次变化时，所有直接使用此 Context 的组件都会重新渲染。当 form 中的控件数量变多时，每次 value 变动都会需要大量的渲染。

FastField 组件的出现就是为了解决这个问题。首先 FastField 组件内部不会直接从 Context 中获取信息，而是靠 context 组件从 props 传递进来。其次组件内部实现了 shouldComponentUpdate 方法来阻止不必要的渲染。

```tsx
class FastFieldInner<Values = {}, Props = {}> extends React.Component<
  FastFieldInnerProps<Values, Props>,
  {}
> {
  // ...
  shouldComponentUpdate(props: FastFieldInnerProps<Values, Props>) {
    if (this.props.shouldUpdate) {
      return this.props.shouldUpdate(props, this.props);
    } else if (
      props.name !== this.props.name ||
      getIn(props.formik.values, this.props.name) !==
        getIn(this.props.formik.values, this.props.name) ||
      getIn(props.formik.errors, this.props.name) !==
        getIn(this.props.formik.errors, this.props.name) ||
      getIn(props.formik.touched, this.props.name) !==
        getIn(this.props.formik.touched, this.props.name) ||
      Object.keys(this.props).length !== Object.keys(props).length ||
      props.formik.isSubmitting !== this.props.formik.isSubmitting
    ) {
      return true;
    } else {
      return false;
    }
  }
  // ...
}

export const FastField = connect<FastFieldAttributes<any>, any>(FastFieldInner);
```

从代码中可以看出，FastField 会比较 name 值、value、error、touched、props 有无增减，以及当前的 form 的状态，来决定是否需要重新渲染。

formik 的 values 不需要固定的格式，在使用时可以动态添加删除 values 中的值。

```ts
function formikReducer<Values>(
  state: FormikState<Values>,
  msg: FormikMessage<Values>
) {
  switch (msg.type) {
    // ...
    case "SET_FIELD_VALUE":
      return {
        ...state,
        values: setIn(state.values, msg.payload.field, msg.payload.value),
      };
    // ...
  }
}
// ...
const getFieldProps = React.useCallback(
  (nameOrOptions): FieldInputProps<any> => {
    // ...
    const valueState = getIn(state.values, name);
    // ...
  },
  [handleBlur, handleChange, state.values]
);
```

关键就在于 setIn 和 getIn 这两个函数。

field 的 name 值其实代表一个路径。比如说下面的这个 values 对象，底层的属性 c 存放着多选 checkbox 的 value，这时这个 checkbox 控件的 name 就是`"a.b.c[0]"`或者`"a.b.c.1"`。

```js
const values = {
  a: {
    b: {
      c: ["v1", "v2"],
    },
  },
};
```

现在我们再看下 getIn 函数的代码，name 值会通过 lodash 的 toPath 方法变成一个 path 数组。延用上面的例子，那么 path 就是`["a", "b", "c", "0"]`，最终可以通过 path 取到目标值。

```js
/**
 * Deeply get a value from an object via its path.
 */
export function getIn(
  obj: any,
  key: string | string[],
  def?: any,
  p: number = 0
) {
  const path = toPath(key);
  while (obj && p < path.length) {
    obj = obj[path[p++]];
  }
  return obj === undefined ? def : obj;
}
```

setIn 函数稍微复杂一点。它也会将 name 转为 path 数组来设置值。如果中间有某些路径是不存在的，就要根据路径是否是 number 类型来判断添加一个`{}`或`[]`。类似于给定一个文件路径来创建文件，路径中不存在的地方自动创建相应的文件夹。其次，在修改属性值的过程中，需要将这个路径上所有的对象或者数组做一个浅拷贝然后替换原来的。这样保证了原先对象的不可变性。

```ts
export function setIn(obj: any, path: string, value: any): any {
  let res: any = clone(obj); // this keeps inheritance when obj is a class
  let resVal: any = res;
  let i = 0;
  let pathArray = toPath(path);

  for (; i < pathArray.length - 1; i++) {
    const currentPath: string = pathArray[i];
    let currentObj: any = getIn(obj, pathArray.slice(0, i + 1));

    if (currentObj && (isObject(currentObj) || Array.isArray(currentObj))) {
      resVal = resVal[currentPath] = clone(currentObj);
    } else {
      const nextPath: string = pathArray[i + 1];
      resVal = resVal[currentPath] =
        isInteger(nextPath) && Number(nextPath) >= 0 ? [] : {};
    }
  }

  // Return original object if new value is the same as current
  if ((i === 0 ? obj : resVal)[pathArray[i]] === value) {
    return obj;
  }

  if (value === undefined) {
    delete resVal[pathArray[i]];
  } else {
    resVal[pathArray[i]] = value;
  }

  // If the path array has a single element, the loop did not run.
  // Deleting on `resVal` had no effect in this scenario, so we delete on the result instead.
  if (i === 0 && value === undefined) {
    delete res[pathArray[i]];
  }

  return res;
}
```

最后来看看 FieldArray 组件，在这个例子中我们可以通过 FieldArray 提供的方法，添加、删除、插入表单。注意 FieldArray 中渲染的 Field 组件的 name 值，代表的是这个值在 values 中的路径。

```tsx
<Formik
  initialValues={{ friends: ["jared", "ian", "brent"] }}
  render={({ values }) => (
    <Form>
      <FieldArray
        name="friends"
        render={(arrayHelpers) => (
          <div>
            {values.friends && values.friends.length > 0 ? (
              values.friends.map((friend, index) => (
                <div key={index}>
                  <Field name={`friends.${index}`} />
                  <button
                    type="button"
                    onClick={() => arrayHelpers.remove(index)} // remove a friend from the list
                  >
                    -
                  </button>
                  <button
                    type="button"
                    onClick={() => arrayHelpers.insert(index, "")} // insert an empty string at a position
                  >
                    +
                  </button>
                </div>
              ))
            ) : (
              <button type="button" onClick={() => arrayHelpers.push("")}>
                {/* show this when user has removed all friends from the list */}
                Add a friend
              </button>
            )}
            <div>
              <button type="submit">Submit</button>
            </div>
          </div>
        )}
      />
    </Form>
  )}
/>
```

我们来看这个 push 操作，内部调用了 updateArrayField 方法。这个方法利用传入的更新函数来更新 state 中的 values、errors 和 touched。而 push 的更新函数就是将原数组浅拷贝，再将新的值放入到新数组的末尾，touched 和 errors 无需变化。

```ts
updateArrayField = (
  fn: Function,
  alterTouched: boolean | Function,
  alterErrors: boolean | Function
) => {
  const {
    name,
    formik: { setFormikState },
  } = this.props;
  setFormikState((prevState: FormikState<any>) => {
    // ...
    let values = setIn(
      prevState.values,
      name,
      fn(getIn(prevState.values, name))
    );

    let fieldError = alterErrors
      ? updateErrors(getIn(prevState.errors, name))
      : undefined;
    let fieldTouched = alterTouched
      ? updateTouched(getIn(prevState.touched, name))
      : undefined;
    // ...
    return {
      ...prevState,
      values,
      errors: alterErrors
        ? setIn(prevState.errors, name, fieldError)
        : prevState.errors,
      touched: alterTouched
        ? setIn(prevState.touched, name, fieldTouched)
        : prevState.touched,
    };
  });
  // ...
};

push = (value: any) =>
  this.updateArrayField(
    (arrayLike: ArrayLike<any>) => [
      ...copyArrayLike(arrayLike),
      cloneDeep(value),
    ],
    false,
    false
  );
```

同理，remove 操作需要浅拷贝 values、errors、touched，然后从中除去某个位置值。而 insert 操作则是将新的值插入到这个位置。

```ts
remove<T>(index: number): T {
	// We need to make sure we also remove relevant pieces of `touched` and `errors`
	let result: any;
	this.updateArrayField(
		// so this gets call 3 times
		(array?: any[]) => {
		const copy = array ? copyArrayLike(array) : [];
		if (!result) {
			result = copy[index];
		}
		if (isFunction(copy.splice)) {
			copy.splice(index, 1);
		}
		return copy;
		},
		true,
		true
	);

	return result as T;
}
// ...
insert = (index: number, value: any) =>
	this.updateArrayField(
		(array: any[]) => insert(array, index, value),
		(array: any[]) => insert(array, index, null),
		(array: any[]) => insert(array, index, null)
	);
```

## Conclusion

formik 减少了大量的模板代码，并且支持动态增减表单项，在性能上也提供了优化版的组件支持大量表单项。足以完成日常工作。限于篇幅，还有不少细节在本文中没有提及，但是核心的代码都有覆盖到。
