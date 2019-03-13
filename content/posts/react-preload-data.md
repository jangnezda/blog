---
title: "Preloading data for React components"
date: 2019-03-12T22:21:59+01:00
draft: false
toc: false
images:
tags: 
  - react
  - redux
---

For every non-trivial web application there is a need to preload some data before rendering the content. For example, to display a list of users we need to fetch it first from the server. React web applications are no exception, so let's take a look how we can preload some data in React components.

## React documentation

I always check the documentation first when exploring how to do something with React. There are hints and recommendations on how to do (or not do) things in React applications. Sure enough, there's a [section](https://reactjs.org/docs/faq-ajax.html) about fetching data from APIs. The recommended way is to use `componentDidMount()` lifecycle method:

```js
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      // some state
    };
  }

  componentDidMount() {
    fetch("https://api.example.com/items")
      .then(res => res.json())
      .then(
        // update the state with results
      );
  }
  
  render() {
    // render the component using data from state
  }
}
```

This works as expected: data is fetched and then rendered. The problem with this solution is that our component contains data loading logic that is coupled to a lifecycle method. This means it's harder to test and reuse the component. Ideally, we'd want to move this logic out and instead inject `items` array as a property into this component. That way, we can easily test it and use it in Storybook, for example.

## Wrapper component

To solve this tight coupling, one may instead move the fetching logic into a wrapper component. This means that we can define a separate component that just does the fetching, then renders `MyComponent` with `items` prop.

```jsx
class MyComponent extends React.Component {
  render() {
    const { items, itemsLoaded } = this.props;
    
    if (itemsLoaded) {
      return (
        // render items
      );
    }
    
    return (
      // render loading indicator
    );
  }
}

class Wrapper extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      items: [],
      itemsLoaded: false
    };
  }
  
  componentDidMount() {
    fetch("https://api.example.com/items")
      .then(res => res.json())
      .then(result => {
        this.setState({
          items: result.items,
          itemsLoaded: true
        });
      });
  }

  render() {
    const { items, itemsLoaded } = this.state;
    
    return (
      <MyComponent
        items={items}
        itemsLoaded={itemsLoaded}
        {...this.props}
      />
    );
  }
}
```

Ok, so now we can use the `Wrapper` component elsewhere in our app's pages and the inner `MyComponent` in tests and Storybook. Great! But we're not satisfied yet, because the pattern of preloading data is common for different components and api calls, and it would be tedious to write specific wrapper components for every use-case.

A more generalized version of wrapper component would accept the data fetching function as a property, then invoke it in `componentDidMount()` and pass the result to the inner component:

```jsx
class MyComponent extends React.Component {
  render() {
    const { data, dataLoaded } = this.props;
    
    if (dataLoaded) {
      return (
        // render items
      );
    }
    
    return (
      // render loading indicator
    );
  }
}

class Wrapper extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      data: null,
      dataLoaded: false
    };
  }
  
  componentDidMount() {
    this.props.preload()
      .then(data => {
        this.setState({
          data,
          dataLoaded: true
        });
      });
  }

  render() {
    const { component: Component } = this.props;
    const { data, dataLoaded } = this.state;
    
    return (
      <Component
        data={data}
        dataLoaded={dataLoaded}
        {...this.props}
      />
    );
  }
}

// ... used elsewhere in the app as:
class MyPage extends React.Component {
  render() {
    return (
      <Wrapper component={MyComponent} preload={actions.fetchItems} />
    );
  }
}

```

This generalized version looks like a good and reusable solution. However, it's still a bit ugly and it's annoying that `MyPage` has to be aware of preloading boilerplate. Wouldn't it be nice if we could encapsulate that within `MyComponent`?

## Higher-Order component

We can reach for another technique that is frequently used in React apps and that's higher-order components. As React documentation [explains](https://reactjs.org/docs/higher-order-components.html), a higher-order component is a function that takes a component and returns a new component. The idea is that instead of having a wrapper component, we define a higher-order component for prefetching and then wrap `MyComponent` with `prefetch()` function. This is what we want to achieve:

```jsx
class MyComponent extends React.Component {
  render() {
    const { data, dataLoaded } = this.props;
    
    if (dataLoaded) {
      return (
        // render items
      );
    }
    
    return (
      // render loading indicator
    );
  }
}

class MyComponentWithData = prefetch({
  onComponentDidMount: actions.fetchItems
})(MyComponent);

// ... used elsewhere in the app as:
class MyPage extends React.Component {
  render() {
    return (
      <MyComponentWithData />
    );
  }
}
```
This is much better. `MyPage` does not care about intricacies of loading data necessary for `MyComponent` rendering. The `prefetch()` function looks quite similar to previous `WrapperComponent`, except that it takes a function as parameter, which is then executed in `componentDidMount()` method, and returns a function which in turn returns a React component:

```jsx
const prefetch = ({ onComponentDidMount }) =>
  WrappedComponent => class extends React.Component {
    constructor(props) {
      super(props);
      this.state = {
        data: null,
        dataLoaded: false
      };
    }
  
    componentDidMount() {
      onComponentDidMount()
        .then(data => {
          this.setState({
            data,
            dataLoaded: true
          });
        });
    }

    render() {
      const { component: Component } = this.props;
      const { data, dataLoaded } = this.state;
    
      return (
        <WrappedComponent
          data={data}
          dataLoaded={dataLoaded}
          {...this.props}
        />
      );
    }
  }
```

We could generalize the `prefetch()` function even more: it could take in other lifecycle methods as well (`componentDidUpdate()`, `componentWillReceiveProps()`, and so on). The function would become somewhat complex, though, due to state management and wrapped components would have to know exactly what state is passed to them as props. It would be quite brittle design. To solve that I recommend to reach for [Redux](https://redux.js.org/), because then the state management isn't baked into components.

As you might guess, the pattern I described in this post isn't something new. There is also a cool little package called [react-lifecycle-component](https://github.com/JamieDixon/react-lifecycle-component), which you can use to solve prefetching data in generic way. Works great with Redux, too.