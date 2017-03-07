#Serverless CMS - Part 2
In our previous proof-of-concept demo, we built a bare bones admin for generating a static website (well, really just a single web page). It was very basic, with only the ability to edit some text on the page and set the site title and description.

For this next demo, we are going to build on our example and add rich text editing and image upload capabilities.

Here are links to the previous article and demo project files:

- [Part 1](https://css-tricks.com/build-custom-cms-serverless-static-site-generator/)
- [Part 1 Demo](https://github.com/johnpolacek/serverless-cms)
- [Part 2 Demo](https://github.com/johnpolacek/serverless-cms-2)



##Rich Text Editing
[TinyMCE](https://www.tinymce.com) is the most widely used web-based Rich Text Editor around, so let’s use it. We can add it to our admin form pretty easily. 

Because it has been around for so long, there are [many configuration options](https://www.tinymce.com/docs/configure/) available for TinyMCE. For this demo, we only need a few of them.

```
tinymce.init({
  selector: '#calloutText',
  menubar: false,
  statusbar: false,
  toolbar: 'undo redo | styleselect | bold italic | link',
  plugins: 'autolink link'
});
```

Because the rich text editor will encode its content as markup, we need to update the JSRender template to output the data value for `calloutText` as html.

```
<div class="jumbotron">
  <div class="container">
  <h1 class="display-3">{{>calloutHeadline}}</h1>
  {{:calloutText}}
  ...
  
```

[Screenshot here]

##Image Uploads

For this next step, we are going to add an image background to our Jumbotron.

First, we need to add a new form field so an admin can select a file to upload, then update our form submit script to upload the image to S3. With multiple uploads and callbacks happening here, we can create an upload helper method and use [Deferred objects](https://api.jquery.com/jquery.when/) to make our ajax calls [run simultaneously](https://css-tricks.com/multiple-simultaneous-ajax-requests-one-callback-jquery/).

```
$('body').on('submit','#form-admin',function(e) {
	e.preventDefault();
	var formData = {};
	var $formFields = $('#form-admin').find('input, textarea, select').not(':input[type=button], :input[type=submit], :input[type=reset]');

	$formFields.each(function() {
		formData[$(this).attr('name')] = $(this).val();
	});
	
	var jumbotronHTML = '<!DOCTYPE html>' + $.render.jumbotronTemplate(formData);
	var fileHTML = new File([jumbotronHTML], 'index.html', {type: "text/html", lastModified: new Date()});
	var fileJSON = new File([JSON.stringify(formData)], 'admin.json');

	var uploadHTML = $.Deferred();
	var uploadJSON = $.Deferred();
	var uploadImage = $.Deferred();
	
	upload({
		Key: 'index.html',
		Body: fileHTML,
		ACL: 'public-read',
		ContentDisposition: 'inline',
		ContentType: 'text/html'
	}, uploadHTML);

	upload({
		Key: 'admin/index.json',
		Body: fileJSON,
		ACL: 'public-read'
	}, uploadJSON);

	if ($('#calloutBackgroundImage').prop('files').length) {
		upload({
			Key: 'img/callout.jpg',
			Body: $('#calloutBackgroundImage').prop('files')[0],
			ACL: 'public-read'
		}, uploadImage);
	} else {
		uploadImage.resolve();
	}	
	
	$.when(uploadHTML, uploadImage, uploadJSON).then(function() {
		$('#form-admin').prepend('<p id="success">Update successful! <a href="../index.html">View Website</a></p>');
	})

});

function upload(uploadData, deferred) {
	s3.upload(uploadData, function(err, data) {
		if (err) {
			return alert('There was an error: ', err.message);
			deferred.reject();
		} else {
			deferred.resolve();
		}
	});
}
```
Next, we update our jumbotron to display the callout background image.

```
.jumbotron {
  background-image: url(../img/callout.jpg);
  background-repeat: no-repeat;
  background-attachment: fixed;
  background-position: center;
  background-size: cover;
}
```
##Blog Posts
Let’s use our new rich text editing and image uploading capabilities together to add blog posts.

We will be doing a lot more templating. We can make this simpler by writing a helper function to automatically register jsx templates.

```
$('script[type="text/x-jsrender"]').each(function() {
  $.templates($(this).attr('id'), '#'+$(this).attr('id'));
});

```

Let’s add navigation to our admin page so we can manage different areas of the site, in this case a blog. We can make a nav bar template partial.

```
<script type="text/x-jsrender" id="adminNav">
  <nav class="navbar navbar-light rounded bg-faded my-4">
    <div class="navbar-collapse" id="navbarNav">
      <ul class="nav navbar-nav d-flex flex-row">
        <li class="nav-item pl-2 pr-3 mr-1 border-right">
          <a class="nav-link text-primary" href="#adminIndex">Admin</a>
        </li>
        <li class="nav-item px-2{{if active=='adminIndex'}} active{{/if}}">
          <a class="nav-link" href="#adminIndex">Home {{if active=='adminIndex'}}<span class="sr-only">(current)</span>{{/if}}</a>
        </li>
        <li class="nav-item px-2{{if active=='adminBlog'}} active{{/if}}">
          <a class="nav-link" href="#adminBlog">Blog {{if active=='adminBlog'}}<span class="sr-only">(current)</span>{{/if}}</a>
        </li>
      </ul>
    </div>
  </nav>
</script>

```
Now we update our existing admin page with the nav bar and a new id.

```
<script type="text/x-jsrender" id="adminHome">
  {{include tmpl='adminNav' /}}
  <form class="py-2" id="form-admin">
    <h3 class="py-2">Site Info</h3>
    ...
    
```
Next we can add navigation to our admin page by creating a helper function that connects the nav button links to our templates. We will be using the rich text editor as we edit different areas of the site, so creating another helper function will enable us to easily configure the editor with different settings.

```
$('body').on('click','.nav-link', function(e) {
  e.preventDefault();
  loadPage($(this).attr('href').slice(1));
});

function loadPage(pageId) {
  adminData = {};
$.getJSON(pageId+'.json', function( data ) {
    adminData = data;
  }).always(function() {
    $('.container').html($.render[pageId]($.extend(adminData,{active:pageId})));
    initRichTextEditor();
  });
}

function initRichTextEditor(settings) {
  tinymce.init($.extend({
    selector:'textarea',
    menubar: false,
    statusbar: false,
    toolbar: 'undo redo | styleselect | bold italic | link',
    plugins: 'autolink link',
    init_instance_callback : function(editor) {
      $('.mce-notification-warning').remove();
    }
  }, settings ? settings : {}));
}
```
Create a new admin section for managing the blog. We don't have any posts saved yet, so we need a button to create one.

```
<script type="text/x-jsrender" id="adminBlog">
  {{include tmpl='adminNav' /}}
  <div id="blogPosts">
    <h3 class="py-2">Blog Posts</h3>
    <button id="newPostButton" class="btn btn-primary">+ New Post</button>
    ...
```
Also we need a form to write these posts. Note that we include a hidden file input which we will use to allow the rich text editor to upload images.

```
<script type="text/x-jsrender" id="editBlogPost">
  <form class="py-2" id="form-blog">
    {{if postTitle}}
      <h3 class="py-2">Edit Blog Post</h3>
    {{else}}
      <h3 class="py-2">New Blog Post</h3>
    {{/if}}
    <div class="form-group">
        <label for="postTitle">Title</label>
        <input type="text" value="{{>postTitle}}" class="form-control" id="postTitle" name="postTitle" />
    </div>
    <div class="form-group pb-2">
      <textarea class="form-control" id="postContent" name="postContent" rows="12">{{>postContent}}</textarea>
    </div>
    <div class="hidden-xs-up">
	   <input type="file" id="imageUploadFile" />
	 </div>
    <div class="text-xs-right">
      <button class="btn btn-link">Cancel</button>
      <button type="submit" class="btn btn-primary">Save</button>
    </div>  
  </form>
</script>
```

Enabling admin to edit multiple pages of the site will require us to structure our site generation differently. Every time a change is made to the site title and info, we need to propogate that to both the homepage and the blog.

First, we create template partials for our site nav that we can include in each of the page templates.

```
<script type="text/x-jsrender" id="siteNav">
  <nav class="navbar navbar-static-top navbar-dark bg-inverse">
    <a class="navbar-brand pr-2" href="#">{{>siteTitle}}</a>
    <ul class="nav navbar-nav">
      <li class="nav-item{{if active=='index'}} active{{/if}}">
        <a class="nav-link" href="{{>navPath}}index.html">Home {{if active=='index'}}<span class="sr-only">(current)</span>{{/if}}</a>
      </li>
      <li class="nav-item{{if active=='blog'}} active{{/if}}">
        <a class="nav-link" href="{{>navPath}}blog.html">Blog {{if active=='blog'}}<span class="sr-only">(current)</span>{{/if}}</a>
      ...
```

```
<body>
  {{include tmpl='siteNav' /}}
  ...
```

When our admin clicks the new post button, they should be presented with our edit form. We can create a function to do just that and attach it to a click event on the button.

Additionally, we want to be able to add images to our blog posts. In order to do that, we need to add a custom image upload window to our rich text editor with some configuration settings.

```
function editPost(postData) {
	$('.container').append($.render.editBlogPost(postData));
	initRichTextEditor({
		toolbar: 'undo redo | styleselect | bold italic | bullist numlist | link addImage',
		setup : function(editor) {
			editor.addButton('addImage', {
				text: 'Add Image',
				icon: false,
				onclick: function() {
					// Open window
					editor.windowManager.open({
						title: 'Add Image',
						body: [{
								type: 'button', name: 'uploadImage', label:'Select an image to upload', text: 'Browse',
								onclick: function(e) {
									$('#imageUploadFile').click();
								}, onPostRender: function() {
									addImageButton = this;
								}
							},{
								type: 'textbox', name: 'imageDescription', label:'Image Description'
							}],
						buttons: [{
								text: 'Cancel',
								onclick: 'close'
							},{
								text: 'OK',
								classes: 'widget btn primary first abs-layout-item',
								disabled: true,
								onclick: 'close',
								id:'addImageButton'
							}]
					});
				}
			});
		}
	});
}

$('body').on('click','#addImageButton',function() {
  if ($(this).hasClass('mce-disabled')) {
    alert('Please select an image');
  } else {
    var fileUploadData,
      extension = 'jpg',
      mimeType = $('#imageUploadFile')[0].files[0].type; // You can get the mime type

    if (mimeType.indexOf('png') != -1) {
      extension = 'png';
    }
    if (mimeType.indexOf('gif') != -1) {
      extension = 'gif';
    }

    var filepath = 'img/blog/'+((new Date().getMonth())+1)+'/'+Date.now()+'.'+extension;
    
    upload({
        Key: filepath,
        Body: $('#imageUploadFile').prop('files')[0],
        ACL: 'public-read'
    }).done(function() {
      var bucketUrl = 'http://serverless-cms.s3-website-us-east-1.amazonaws.com/';
      tinyMCE.activeEditor.execCommand('mceInsertRawHTML', false, '<p><img src="'+bucketUrl+filepath+'" alt="'+$('.mce-textbox').val()+'" /></p>');
      $('#imageUploadFile').val();
      tinyMCE.activeEditor.windowManager.close();
    });
  }
});
```

The above code will add a special ‘Add Image’ button to the rich text editor controls which can open a modal where an admin can choose an image to upload. When admin clicks ‘Browse’, we have added a click trigger to the hidden file input in the edit form.

Once they have selected an image to add to the post, clicking ‘OK’ to close the window will also upload the image to S3 then insert the image at the cursor location in the rich text editor.

Next, we need to save the blog post. To accomplish this, we will combine our form data with a template to render html and upload to S3. There is a template for the blog and the individual post itself. We also need to store the post data so that the admin can see a list of posts and make edits.

```
$('body').on('submit','#form-blog',function(e) {
  e.preventDefault();

  var updateBlogPosts = $.Deferred();
  if ($(this).attr('data-post-id') === '') {
    postId = Date.now();
  } else {
    postId = $(this).attr('data-post-id');
  }

  if (!adminData.posts) {
    adminData.posts = [];
  }

  var postUrl = 'posts/'+($('#title').val().toLowerCase().replace(/[^\w\s]/gi, '').replace(/\s/g,'-'))+'.html';
  var postTitle = $('#title').val();
  var postContent = tinyMCE.activeEditor.getContent({format:'raw'});

  adminData.posts.push({
    url: postUrl,
    title: postTitle,
    excerpt: $(postContent)[0].innerText
  });

  var uploads = generateHTMLUploads('blog');
  uploads.push(generateAdminDataUpload());
  
  var postHTML = '<!DOCTYPE html>' + $.render['blogPostTemplate']($.extend(adminData,{
    active: 'blog',
    title: postTitle,
    content: postContent,
    navPath: '../'
  }));
  
  var fileHTML = new File([postHTML], postUrl, {type: "text/html", lastModified: new Date()});
  uploads.push(upload({
        Key: postUrl,
        Body: postHTML,
        ACL: 'public-read',
        ContentDisposition: 'inline',
        ContentType: 'text/html'
    })
  )

  $.when.apply($, uploads).then(function() {
    loadPage('adminBlog');
  });

});
```
In our admin blog page, we will list our published posts.

```
<script type="text/x-jsrender" id="adminBlog">
  {{include tmpl='adminNav' /}}
  <div id="blogPosts">
    <h3 class="py-2">Blog Posts</h3>
    <button id="newPostButton" class="btn btn-primary my-1">+ New Post</button>
    {{if posts}}
      <div class="container p-0">
        <ul class="list-group d-inline-block">
          {{for posts}}
            <li class="list-group-item">
              <span class="pr-3">{{>title}}</span>
              <a href="../{{>url}}" target="_blank" class="pl-3 float-xs-right">view</a>
              <a href="#" data-id="{{:#getIndex()}}" data-url="{{>url}}" class="edit-post pl-3 float-xs-right">edit</a>
            </li>
          {{/for}}
        </ul>
      </div>
    {{/if}}
  </div>
</script>
```

Finally, we will expand our new post click handler to handle editing posts by loading the post data into the form template.

```
$('body').on('click','#newPostButton,.edit-post',function(e) {
  e.preventDefault();
  $('#blogPosts').remove();
  if ($(this).is('#newPostButton')) {
    editPost({});
  } else {
    var postId = $(this).attr('data-id');
    var postUrl = $(this).attr('data-url');

    $('<div />').load('../'+postUrl, function() {
      editPost({
        id:postId,
        title:$(this).find('h1').text(),
        content:$(this).find('#content').html()
      });
    });
  }
});
```

## Next Steps

Obviously this is a basic example and is missing a lot of key functionality, like the ability to have draft posts, delete blog posts, or pagination.

Additionally, as the site grows in scope, generating batches of html files on the client side will become burdensome and unreliable. However, we can keep our architecture serverless by offloading the site generation to [AWS Lambda](https://aws.amazon.com/lambda/) and create microservices for updating site info and managing blog posts.

Managing our site data structure by updating flat json files stored on S3 is inexpensive and can lend itself to easily setting up backups and restoration. However, for projects that are more than a simple blog or marketing site, it is possible to use [AWS Dynamo DB](https://aws.amazon.com/dynamodb/) to store data, which is also supported by the [AWS SDK for JavaScript in the Browser](https://aws.amazon.com/sdk-for-browser/).