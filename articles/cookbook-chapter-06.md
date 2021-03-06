- [Chapter 06 - The form page](#sec-1)

# Chapter 06 - The form page<a id="sec-1"></a>

For the listing page, we took a straightforward approach by using the `Doc.Async` function to render the content of the async block.

For the form page, we are going the take a different approach, as the page will send data to the server and wait for its response. While waiting for the response, we want it to display a spinner component to instruct the user to wait for the request to finish.

![img](./images/cookbook-chapter-06-image-01.png "The form page")

Also, the `SaveUser` server function will return a `Result` type, so we can handle any form validation.

Let's start by adding two new RPC functions to the ***Server.fs*** file and some validation functions:

```fsharp
...
let private optionToResult msg o =
    match o with
    | None -> Result.Error msg
    | Some v -> Result.Ok v

let private validateFirstname (user:User) =
    match user.Firstname with
    | null -> Error "No fistname found."
    | "" -> Error "Fistname is empty."
    | _ -> Ok user

let private validateLastname (user:User) =
    match user.Firstname with
    | null -> Error "No lastname found."
    | "" -> Error "Lastname is empty."
    | _ -> Ok user

let private validateRequest userResult =
    userResult
    |> Result.bind validateFirstname
    |> Result.bind validateLastname

let private updateUser user =
    { user with
          UpdateDate = DateTime.Now
    }

[<Rpc>]
let SaveUser (dto:User) : Async<Result<User,string>> =
    async {
        return
            dto
            |> Ok
            |> validateRequest
            |> Result.map updateUser
    }

[<Rpc>]
let GetUser (code:int64) : Async<Result<User,string>> =
    async {
        // simulate delay
        do! Async.Sleep(2000)

        let userO =
            dbUsers()
            |> List.tryFind(fun u -> u.Code = code)
            |> optionToResult "User not found!"

        return userO
    }

```

The `GetUser` function tries to retrieve the user data by its id (code) and the `SaveUser` functions performs some input validation and update the `CreateDate` field, if everything is ok.

Notice that both functions are returning a `Result` type and WebSharper compiler can deal with it transparently for us.

Finally, let's add two more files to the project:

-   Page.Form.fs
-   templates/Page.Form.html

This is the HTML template:

```html
<form>
    <replace ws-replace="AlertBox"></replace>

    <div class="form-group row">
        <label for="code" class="col-sm-2 col-form-label">Code:</label>
        <div class="col-sm-10">
            <input type="text" readonly class="form-control-plaintext" id="code" ws-var="Code">
        </div>
    </div>

    <div class="form-group row">
        <label for="firstname" class="col-sm-2 col-form-label">Firstname</label>
        <div class="col-sm-10">
            <input type="text" class="form-control" id="firstname" ws-var="Firstname">
        </div>
    </div>

    <div class="form-group row">
        <label for="lastname" class="col-sm-2 col-form-label">Lastname</label>
        <div class="col-sm-10">
            <input type="text" class="form-control" id="lastname" ws-var="Lastname">
        </div>
    </div>

    <div class="form-group row">
        <label for="updated-at" class="col-sm-2 col-form-label">Updated At:</label>
        <div class="col-sm-10">
            <input type="text" readonly class="form-control-plaintext" id="updated-at" ws-var="UpdatedAt">
        </div>
    </div>

    <div class="col-12">
        <button type="button" class="btn btn-primary" ws-onclick="OnSave">save</button>
        <button type="button" class="btn btn-link" ws-onclick="OnBack">back</button>
    </div>
</form>

```

This is pretty similar to the Listing page. It uses the `ws-var` attributes for editable content. They are used by *Reactive Variables* and allows two-way data binding.

The code for the ***Page.Form.fs*** file is:

```fsharp
namespace WebSharperTutorial.FrontEnd.Pages

open WebSharper
open WebSharper.UI
open WebSharper.UI.Client
open WebSharper.UI.Html
open WebSharper.JavaScript

open WebSharperTutorial.FrontEnd
open WebSharperTutorial.FrontEnd.Components

[<JavaScript>]
module PageForm =

    type private formTemplate = Templating.Template<"templates/Page.Form.html">

    let private AlertBox (rvStatusMsg:Var<string option>) =
        rvStatusMsg.View
        |> View.Map (fun msgO ->
            match msgO with
            | None ->
                Doc.Empty
            | Some msg ->
                div [ attr.``class`` "alert alert-primary"
                      Attr.Create "role" "alert"
                    ]
                    [ text msg ]
        )
        |> Doc.EmbedView

    let private spinner msg =
      div
        [ attr.``class`` "spinner-border text-warning"
          Attr.Create "role" "status"
        ]
        [ span [ attr.``class`` "sr-only" ] [ text msg ]
        ]

    let private frameContent navBar content =
        [
            navBar
            div [ attr.``class`` "container" ]
                [
                  div [ attr.``class`` "row" ]
                      [ div [ attr.``class`` "col-12" ]
                            [ content ]
                      ]
                ]
        ]
        |> Doc.Concat

    let Main router code =
        let rvStatusMsg = Var.Create None
        let statusMsgBox = AlertBox rvStatusMsg

        let rvModel = Var.CreateWaiting<DTO.User>()
        let submitter =
            Submitter.CreateOption<DTO.User> rvModel.View

        let loadModel() =
            async {
                let! modelR =
                    Server.GetUser code

                match modelR with
                | Error error ->
                    Var.Set rvStatusMsg (Some error)

                | Ok model ->
                    Var.Set rvModel model
                    submitter.Trigger()

                return ()
            }

        let navBar =
            NavigationBar.Main router

        let content =
            submitter.View
            |> View.Map (fun modelO ->
                match modelO with
                | None -> spinner "loading..."
                | Some model ->
                    let rvCode =
                         rvModel.Lens
                             (fun model -> string model.Code)
                             (fun model value -> { model with Code = int64 value })

                    formTemplate()
                        .AlertBox(statusMsgBox)
                        .Code(rvCode)
                        .Firstname(Lens(rvModel.V.Firstname))
                        .Lastname(Lens(rvModel.V.Lastname))
                        .UpdatedAt(model.UpdateDate.ToShortDateString())
                        .OnSave(fun evt ->
                            async {
                                let! modelR =
                                    Server.SaveUser rvModel.Value

                                match modelR with
                                | Error error ->
                                    Var.Set rvStatusMsg (Some error)

                                | Ok model ->
                                    Var.Set rvModel model
                                    Var.Set rvStatusMsg (Some "Saved!")
                                    submitter.Trigger()
                            }
                            |> Async.Start
                      )
                      .OnBack(fun _ ->
                          Var.Set router Routes.Listing
                      )
                      .Doc()

            )
            |> Doc.EmbedView

        loadModel()
        |> Async.Start

        frameContent navBar content

```

You will find a few helper functions to build the alert box and a spinner and you might want to move them to specialized components, as they are starting to repeat.

The main function brings a new approach to render the page. Instead of relying on the `Doc.Async` function as we did before, this function makes use of the `Submitter` type.

Also, worth noting that it uses `Var.CreateWaiting` function. This constructor allows to declare a *Reactive Variable* without initializing it. Later, once the data is ready, we set its value using the `Var.Set` function as usual.

> Tip: if you forget to set the value of *Reactive Variable* created with the `Var.CreateWaiting` function, your page won't render.

The `Submitter` is linked to the Reactive Variable's View and provides a `Trigger` function. When called, this function will get the latest value from the underlying View (the `rvModel` one) and update itself.

The form content is render by the Submitter View, as highlighted in the code snippet below:

```fsharp
...
let content =
    submitter.View
    |> View.Map (fun modelO ->
        match modelO with
        | None -> spinner "loading..."
        | Some model ->
....
```

If the model's *Reactive Variable* is not ready, the `None` case will be matched and a spinner wheel will be displayed. Once done, the `Some` case will render the form's content.

One last feature worth highlighting is the use of `Lens` function. There are two constructors in the code.

The first one uses getter/setter functions to map the `mode.Code` field back and forth from string and int64. We need this, as the `ws-var` attribute in the HTML template only accepts the `Var<string>` type.

```fsharp
...
| Some model ->
    let rvCode =
          rvModel.Lens
              (fun model -> string model.Code)
              (fun model value -> { model with Code = int64 value })
...

```

The second Lens' construtor is using a the **V Shorthand** and automatically lenses the chosen field for us:

```fsharp
...
.Firstname(Lens(rvModel.V.Firstname))
...

```

The last change we need is to handle the **Form EndPoint** at the ***Main.fs*** file, by updating the `RouteClientPage` function. This is the final version:

```fsharp
[<JavaScript>]
let RouteClientPage () =
    let router = Routes.InstallRouter ()

    router.View
    |> View.Map (fun endpoint ->
        match endpoint with
        | EndPoint.Home ->
            PageHome.Main router

        | EndPoint.Login ->
            PageLogin.Main router

        | EndPoint.Listing ->
            PageListing.Main router

        | EndPoint.Form code ->
            PageForm.Main router code

        | _ ->
            div [] [ text "implementation pending" ]
    )
    |> Doc.EmbedView

```

That's it! This project has room for many improvements, but I will left this task for the reader.

WebSharper is a very powerful framework and there are several features we didn't cover on this tutorial.

Hopefully, it helped you to get started to WebSharper framework and from here, you can take a look at the official documentation.

| [previous](./cookbook-chapter-05.md) | [up](../README.md) |
