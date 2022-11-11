# Hotwire: Turbo Frame-powered Inline Editing

[![Deploy to Heroku](https://www.herokucdn.com/deploy/button.png)][heroku-deploy-app]

[heroku-deploy-app]: https://heroku.com/deploy?template=https://github.com/thoughtbot/hotwire-example-template/tree/hotwire-example-inline-edit

When reading a branch's source code, read the changes commit-by-commit either on
the branch comparison page (for example,
[main...hotwire-example-inline-edit][]), the branch's commits page (for
example, [hotwire-example-inline-edit][]), or the branch's `README.md` file
(for example, [hotwire-example-inline-edit][README]).

[main...hotwire-example-inline-edit]: https://github.com/thoughtbot/hotwire-example-template/compare/hotwire-example-inline-edit
[hotwire-example-inline-edit]: https://github.com/thoughtbot/hotwire-example-template/commits/hotwire-example-inline-edit
[README]: https://github.com/thoughtbot/hotwire-example-template/blob/hotwire-example-inline-edit/README.md

## Our starting point

`Article` records are categorized by `Category` records.

```ruby
# app/models/article.rb

class Article < ApplicationRecord
  has_many :categorizations
  has_many :categories, through: :categorizations

  has_rich_text :content

  with_options presence: true do
    validates :byline
    validates :content
    validates :name
  end
end

# app/models/category.rb

class Category < ApplicationRecord
  has_many :categorizations
  has_many :articles, through: :categorizations
end

# app/models/categorization.rb

class Categorization < ApplicationRecord
  belongs_to :article
  belongs_to :category
end
```

Records are served by a bog-standard `ArticlesController`. We'll be spending
most of our time in the templates for the `show` and `edit` actions.

```ruby
# app/controllers/articles_controller.rb

class ArticlesController < ApplicationController
  def index
    @articles = Article.all
  end

  def show
    @article = Article.find params[:id]
  end

  def edit
    @article = Article.find params[:id]
  end

  def update
    @article = Article.find params[:id]

    if @article.update article_params
      redirect_to article_url(@article)
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private

  def article_params
    params.require(:article).permit(
      :byline,
      :content,
      :name,
      :published_on,
      category_ids: []
    )
  end
end
```

The `app/views/articles/show.html.erb` template renders and formats the
`Article` record's values:

```erb
<%# app/views/articles/show.html.erb %>

<section class="grid gap-2 max-w-prose m-auto">
  <%= link_to "Edit Article", edit_article_path(@article) %>

  <h1><%= @article.name %></h1>

  <span>By: <%= @article.byline %></span>

  <% if @article.published_on.nil? %>
    <span>(Unpublished)</span>
  <% else %>
    <%= localize @article.published_on, format: :long %>
  <% end %>

  <strong>Categories</strong>

  <span>
    <% @article.categories.each do |category| %>
      <span><%= category.name %></span>
    <% end %>
  </span>

  <%= @article.content %>
</section>
```

<img src="https://images.thoughtbot.com/blog-vellum-image-uploads/spw8TIeuTFbCkdumaUUO_article-show.png"
     alt="An Article page with name, byline, published on date, categories, and content"
     height="720">

The `app/views/articles/edit.html.erb` renders a collection of form fields to
change the `Article` record's values. The template's `<form>` element submits a
`PATCH /articles/:id` request when submitted:

```erb
<%# app/views/articles/edit.html.erb %>

<%= form_with model: @article, class: "grid gap-2 max-w-prose m-auto" do |form| %>
  <%= link_to "Back", article_path(@article) %>

  <%= form.label :name %>
  <%= form.text_field :name %>

  <%= form.label :byline %>
  <%= form.text_field :byline %>

  <%= form.label :published_on %>
  <%= form.date_field :published_on %>

  <fieldset>
    <legend>
      <%= @article.class.human_attribute_name(:category_ids) %>
    </legend>

    <%= form.collection_check_boxes :category_ids, Category.all, :id, :name do |builder| %>
      <%= builder.check_box %>
      <%= builder.label %>
    <% end %>
  </fieldset>

  <%= form.label :content %>
  <%= form.rich_text_area :content %>

  <%= form.button %>
<% end %>
```

<img src="https://images.thoughtbot.com/blog-vellum-image-uploads/MniJocQJeKY5nHwCXocw_article-edit.png"
     alt="A fields to edit an Article's name, byline, published on date, categories, and content"
     height="720">

## Editing the Article's name

We'll start by adding support for editing a single attribute first: the
`Article` record's `name`. We'll wrap the `name` portion of the
`app/views/articles/edit.html.erb` within a `<turbo-frame>` element, so that
other pages can load it as an HTML fragment:

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
   <%= link_to "Back", article_path(@article) %>

+  <turbo-frame>
     <%= form.label :name %>
     <%= form.text_field :name %>
+  </turbo-frame>
```

We'll derive the `<turbo-frame>` element's `[id]` by passing the record and a
`"name_turbo_frame"` prefix to the [`dom_id`][dom_id] view helper, then we'll
assign it to a template-local variable named `frame_id`, then render that value
as the `[id]` attribute:

[dom_id]: https://edgeapi.rubyonrails.org/classes/ActionView/RecordIdentifier.html#method-i-dom_id

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
   <%= link_to "Back", article_path(@article) %>

+  <% frame_id = dom_id(@article, "name_turbo_frame") %>
+
-  <turbo-frame>
+  <turbo-frame id="<%= frame_id %>">
     <%= form.label :name %>
     <%= form.text_field :name %>
   </turbo-frame>
```

Prior to its introduction, the contents of the `<turbo-frame>` element were
participating in a [grid][] layout. Introducing a single element where there
were once two sibling elements changes how the `<label>` and `<input>` elements
occupy available space. We'll apply the [display: contents][contents-rule] to
the `<turbo-frame>` (through Tailwind's [`.contents`][tw-contents] utility
class) so that its descendants continue to participate in the grid layout:

[grid]: https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Grid_Layout
[contents-rule]: https://developer.mozilla.org/en-US/docs/Web/CSS/display#box
[tw-contents]: https://tailwindcss.com/docs/display#contents

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
-  <turbo-frame id="<%= frame_id %>">
+  <turbo-frame id="<%= frame_id %>" class="contents">
     <%= form.label :name %>
     <%= form.text_field :name %>
   </turbo-frame>
```

Next, we'll switch to the `app/views/articles/show.html.erb` template, and wrap
the `Article` record's `name` in a `<turbo-frame>` with a matching `[id]`:

```diff
--- a/app/views/articles/show.html.erb
+++ b/app/views/articles/show.html.erb
   <%= link_to "Edit Article", edit_article_path(@article) %>

+  <% frame_id = dom_id(@article, "name_turbo_frame") %>
+
+  <turbo-frame id="<%= frame_id %>">
     <h1><%= @article.name %></h1>
+  </turbo-frame>
```

We'll also apply the `.contents` class to the `<turbo-frame>` element, so that
its descendants participate in the page's grid-based layout:

```diff
--- a/app/views/articles/show.html.erb
+++ b/app/views/articles/show.html.erb
-  <turbo-frame id="<%= frame_id %>">
+  <turbo-frame id="<%= frame_id %>" class="contents">
     <h1><%= @article.name %></h1>
   </turbo-frame>
```

With matching `<turbo-frame>` elements between the two templates, we can
navigate between the pages by rendering descendant `<a>` elements. We'll render
an "Edit Name" `<a>` element from the `app/views/articles/show.html.erb`
template to navigate the frame to the `ArticlesController#edit` route:

```diff
--- a/app/views/articles/show.html.erb
+++ b/app/views/articles/show.html.erb
   <% frame_id = dom_id(@article, "name_turbo_frame") %>

   <turbo-frame id="<%= frame_id %>" class="contents">
     <h1><%= @article.name %></h1>
+
+    <%= link_to edit_article_path(@article) do %>
+      Edit <%= @article.class.human_attribute_name(:name) %>
+    <% end %>
   </turbo-frame>
```

Then we'll render a "Cancel" `<a>` element to navigate back to the
`ArticlesController#show` route:

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
   <% frame_id = dom_id(@article, "name_turbo_frame") %>

   <turbo-frame id="<%= frame_id %>" class="contents">
     <%= form.label :name %>
     <%= form.text_field :name %>
+
+    <%= link_to "Cancel", article_path(@article) %>
   </turbo-frame>
```

With those links in place, visitors can load and unload the `name` fields, but
cannot save any changes:

https://user-images.githubusercontent.com/2575027/151727236-4d14feeb-cc18-4c2f-9d89-e23f0d9f8a48.mov

## Saving the Article's name

When we navigate the frame to the `app/views/articles/show.html.erb` template,
we'll need a way to submit the changes. We'll render a `<button>` element
alongside the `name` field, making sure to nest it within the `<turbo-frame>`
element:

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
 <%= form_with model: @article, class: "grid gap-2 max-w-prose m-auto" do |form| %>
   <turbo-frame id="<%= frame_id %>" class="contents">
     <%= form.label :name %>
     <%= form.text_field :name %>
+
+    <%= form.button %>
     <%= link_to "Cancel", article_path(@article) %>
   </turbo-frame>
```

Unfortunately, this form submission mechanism is one-sided. While the
`app/views/articles/edit.html.erb` template renders the `<button>` element
nested within a `<form>` element, the matching `<turbo-frame>` element that
loads the fields from the `app/views/articles/show.html.erb` template _is not_
nested within a `<form>`.

We'll render an `app/views/articles/show.html.erb`-side `<form>` element as an
ancestor to the `<turbo-frame>`, then target it by declaring a
`[data-turbo-frame]` attribute to match the `<turbo-frame>` element's `[id]`
attribute:

```diff
   <% frame_id = dom_id(@article, "name_turbo_frame") %>

+  <%= form_with model: @article, class: "contents", data: { turbo_frame: frame_id } do |form| %>
     <turbo-frame id="<%= frame_id %>" class="contents">
       <h1><%= @article.name %></h1>

       <%= link_to edit_article_path(@article) do %>
         Edit <%= @article.class.human_attribute_name(:name) %>
       <% end %>
    </turbo-frame>
+  <% end %>
```

https://user-images.githubusercontent.com/2575027/151736170-938f0552-1546-41b5-a5cf-7d15fde0024f.mov

### Hiding inline actions

While it's crucial to present the "Save" and "Cancel" actions when they're
loaded into the `app/views/articles/show.html.erb` page's `<turbo-frame>`
element, it's as important to hide them when they're rendered as part of the
`app/views/articles/edit.html.erb` template.

If our application were styled in more traditional CSS manner, we'd control
their visibility with cascading CSS rules. For example, consider these two CSS
classes:

```css
             .inline-action { display: none; }
.inline-edit .inline-action { display: initial; }
```

By default, the actions would be hidden. The precedence of the combined
`.inline-edit .inline-action` rule would outweigh the `.inline-action` rule, so
nesting the actions within an `.inline-edit` element would reveal them. We'd
mark the `<turbo-frame>` element (rendered in the
`app/views/articles/show.html.erb` template) with the `.inline-edit` class and
mark the "Cancel" `<a>` element and "Save" `<button>` (rendered in the
`app/views/articles/edit.html.erb` template) with the `.inline-action` class.

Since our sample code styles its elements with Tailwind's [utility-first][tw]
classes, there's an opportunity to introduce a [custom variant][] in the style
of Tailwind's [`group:` variant][group] to achieve the same result:

[tw]: https://tailwindcss.com/docs/utility-first
[custom variant]: https://v2.tailwindcss.com/docs/plugins#adding-variants
[group]: https://tailwindcss.com/docs/hover-focus-and-other-states#styling-based-on-parent-state

```diff
--- a/app/javascript/tailwind.config.js
+++ b/app/javascript/tailwind.config.js
 tailwind.config = {
   corePlugins: {
     preflight: false,
-  }
+  },
+  plugins: [
+    tailwind.plugin(function({ addVariant }) {
+      addVariant("group-inline-edit", ".group.inline-edit &")
+    })
+  ]
 }
```

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
     <%= form.label :name %>
     <%= form.text_field :name %>

-    <%= form.button do %>
+    <%= form.button class: "hidden group-inline-edit:inline" do %>
       Save <%= @article.class.human_attribute_name(:name) %>
     <% end %>
-    <%= link_to "Cancel", article_path(@article) %>
+    <%= link_to "Cancel", article_path(@article), class: "hidden group-inline-edit:inline" %>
   </turbo-frame>
```

```diff
--- a/app/views/articles/show.html.erb
+++ b/app/views/articles/show.html.erb
   <%= form_with model: @article, class: "contents", data: { turbo_frame: frame_id } do %>
-    <turbo-frame id="<%= frame_id %>" class="contents">
+    <turbo-frame id="<%= frame_id %>" class="contents group inline-edit">
       <h1><%= @article.name %></h1>
```

## Extracting and abstracting: view partials

```erb
<%# app/views/articles/_inline_edit.html.erb %>

<% frame_id = dom_id(model, "#{method}_turbo_frame") %>

<%= form_with model: model, data: { turbo_frame: frame_id } do %>
  <turbo-frame id="<%= frame_id %>" class="contents group inline-edit">
    <%= yield %>

    <%= link_to edit_article_path(model) do %>
      Edit <%= model.class.human_attribute_name(method) %>
    <% end %>
  </turbo-frame>
<% end %>
```

```diff
--- a/app/views/articles/show.html.erb
+++ b/app/views/articles/show.html.erb
-  <% frame_id = dom_id(@article, "name_turbo_frame") %>
-
-  <%= form_with model: @article, class: "contents", data: { turbo_frame: frame_id } do %>
-    <turbo-frame id="<%= frame_id %>" class="contents group inline-edit">
+  <%= render "inline_edit", model: @article, method: :name do %>
     <h1><%= @article.name %></h1>
-
-      <%= link_to edit_article_path(@article) do %>
-        Edit <%= @article.class.human_attribute_name(:name) %>
-      <% end %>
-    </turbo-frame>
   <% end %>
```

```erb
<%# app/views/articles/_inline_fields.html.erb %>

<% frame_id = dom_id(form.object, "#{method}_turbo_frame") %>

<turbo-frame id="<%= frame_id %>" class="contents">
  <%= yield %>

  <%= form.button class: "hidden group-inline-edit:inline" do %>
    Save <%= form.object.class.human_attribute_name(method) %>
  <% end %>
  <%= link_to "Cancel", article_path(form.object), class: "hidden group-inline-edit:inline" %>
</turbo-frame>
```

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
-  <% frame_id = dom_id(@article, "name_turbo_frame") %>
-
-  <turbo-frame id="<%= frame_id %>" class="contents">
+  <%= render "inline_fields", form: form, method: :name do %>
     <%= form.label :name %>
     <%= form.text_field :name %>
-
-    <%= form.button class: "hidden group-inline-edit:inline" do %>
-      Save <%= @article.class.human_attribute_name(:name) %>
   <% end %>
-    <%= link_to "Cancel", article_path(@article), class: "hidden group-inline-edit:inline" %>
-  </turbo-frame>
```

## Editing the other fields

With our re-usable view partials, we can wrap the rest of the
`app/views/articles/show.html.erb` template's content in calls to render the
`inline_edit` view partial, passing along the `@article` reference and the name
of the method that backs the block's content:

```diff
--- a/app/views/articles/show.html.erb
+++ b/app/views/articles/show.html.erb
+  <%= render "inline_edit", model: @article, method: :byline do %>
     <span>By: <%= @article.byline %></span>
+  <% end %>

+  <%= render "inline_edit", model: @article, method: :published_on do %>
     <% if @article.published_on.nil? %>
       <span>(Unpublished)</span>
     <% else %>
       <%= localize @article.published_on, format: :long %>
     <% end %>
+  <% end %>

+  <%= render "inline_edit", model: @article, method: :category_ids do %>
     <strong>Categories</strong>

     <span>
       <% @article.categories.each do |category| %>
         <span><%= category.name %></span>
       <% end %>
     </span>
+  <% end %>

+  <%= render "inline_edit", model: @article, method: :content do %>
     <%= @article.content %>
+  <% end %>
 </section>
```

Next, we'll wrap the corresponding content in the
`app/views/articles/edit.html.erb` template, making sure to assign the `[form]`
attribute by passing along the `form: form.id` option to calls to render form
fields:

```diff
--- a/app/views/articles/edit.html.erb
+++ b/app/views/articles/edit.html.erb
+  <%= render "inline_fields", form: form, method: :byline do %>
     <%= form.label :byline %>
     <%= form.text_field :byline %>
+  <% end %>

+  <%= render "inline_fields", form: form, method: :published_on do %>
     <%= form.label :published_on %>
     <%= form.date_field :published_on %>
+  <% end %>

+  <%= render "inline_fields", form: form, method: :category_ids do %>
     <fieldset>
       <legend>
         <%= @article.class.human_attribute_name(:category_ids) %>
       </legend>

       <%= form.collection_check_boxes :category_ids, Category.all, :id, :name do |builder| %>
         <%= builder.check_box %>
         <%= builder.label %>
       <% end %>
     </fieldset>
+  <% end %>

+  <%= render "inline_fields", form: form, method: :content do %>
     <%= form.label :content %>
     <%= form.rich_text_area :content %>
+  <% end %>
```

## Wrapping up

The partials are nested within the `app/views/articles` directory, and make use
of `Article`-specific routing helpers. If we wanted to generalize this pattern
to work with other models, we could declare them within `app/views/application`,
and use the [polymorphic][polymorphic-helpers] variations of the `article_path`
and `edit_article_path` route helpers:

[polymorphic-helpers]: https://edgeapi.rubyonrails.org/classes/ActionDispatch/Routing/PolymorphicRoutes.html

```diff
--- a/app/views/articles/_inline_edit.html.erb
+++ b/app/views/application/_inline_edit.html.erb
   <turbo-frame id="<%= frame_id %>" class="contents group inline-edit">
     <%= yield %>

-    <%= link_to edit_article_path(model) do %>
+    <%= link_to edit_polymorphic_path(model) do %>
       Edit <%= model.class.human_attribute_name(method) %>
     <% end %>
   </turbo-frame>
```

```diff
--- a/app/views/articles/_inline_fields.html.erb
+++ b/app/views/application/_inline_fields.html.erb
   <%= form.button class: "hidden group-inline-edit:inline" do %>
     Save <%= form.object.class.human_attribute_name(method) %>
   <% end %>
-  <%= link_to "Cancel", article_path(form.object), class: "hidden group-inline-edit:inline" %>
+  <%= link_to "Cancel", polymorphic_path(form.object), class: "hidden group-inline-edit:inline" %>
 </turbo-frame>
```
