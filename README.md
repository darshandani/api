<?php

namespace App\Http\Controllers;

use App\Models\Registers;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Validator;



class UserController extends ApiController
{
   
    public function store(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name' => 'required|string',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:8',
        ]);
        if ($validator->fails()) {
            return response()->json(['errors' => $validator->errors()], 422);
        }

        $user = Registers::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        return $this->respondWithSuccess("register sucess");
    }

    /**
     * Display the specified resource.
     */
    public function login(Request $request)
    {
        $inputdata = $request->all();

        $credentials = [
            'email' => $request->email,
            'password' => $request->password,

        ];

        if (auth()->attempt($credentials)) {
            $data = auth()->user();

            $data->token = auth()->user()->createToken('darshan')->accessToken;


            $data->makeHidden('updated_at');
            $data->makeHidden('created_at');


            return $this->respondWithData($data, NULL);
        } else {
            return $this->respondWithError(['Error! Invalid Username or Password']);
        }
    }
    public function logout(Request $res)
    {

        $token = auth()?->user()?->token();
        $token?->revoke();
        return $this->respondWithSuccess('Logout successfully.');
    }
}
<?php

namespace App\Http\Controllers;

use App\Http\Requests\CatagoeryRequest;
use App\Models\Post;
use App\Models\PostTag;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Validator;
use Str;

class PostController extends ApiController
{
    public function PostStore(CatagoeryRequest $request)
    {
        $validator = Validator::make($request->all(), [
            'title' => 'required|string',
            'content' => 'required|string',
            'image' => 'required|image|mimes:jpeg,png,jpg,gif,svg|max:2048',
            "tag_id "    => "array",

        ]);

        if ($validator->fails()) {
            return response()->json(['errors' => $validator->errors()], 422);
        }

        $userId = Auth::check() ? Auth::id() : null;
        $imageArr = array();

        if (isset($request->tag_id)) {
            $imageArr = $request->tag_id;
            unset($request->tag_id);
        }

        $post = new Post();
        $post->user_id = $userId;
        $post->title = $request->title;
        $post->content = $request->content;
        $post->slug = Str::slug($request->title);
        $post->category_id = $request->category_id;

        if ($request->hasFile('image')) {
            $image = $request->file('image');
            $name = time() . '.' . $image->getClientOriginalExtension();
            $destinationPath = public_path('/images');
            $image->move($destinationPath, $name);
            $post->image = 'images/' . $name;
        }

        $post->save();

        if ($request->has('tag_id')) {
            $tag_ids = is_array($request->input('tag_id')) ? $request->input('tag_id') : [$request->input('tag_id')];
            $post->tags()->sync($tag_ids);
        }

        return response()->json(['message' => 'Post created successfully', 'post' => $post], 201);
    }

    public function getPost($id)
    {
        $post = Post::find($id);
        if (!$post) {
            return response()->json(['error' => 'Post not found'], 404);
        }
        return response()->json(['post' => $post]);
    }
    public function getallPost()
    {
        $post = Post::get();
        if (!$post) {
            return response()->json(['error' => 'Post not found'], 404);
        }else{
           
            foreach ($post as $key => $value) {
                $value->category_name = $value->category->name;
                if($value->image){
                    $value->image = asset('public/images/' . $value->image);
                }else{
                    $value->image = asset('public/images/default.jpg');
                }
                
               
                if($value->tags->count()){
                    foreach ($value->tags as $key => $tag) {

                        //dd($tag->name);
                        $tag->makeHidden('slug');
                        $tag->makeHidden('status');
                        $tag->makeHidden('status');
                        $tag->makeHidden('pivot');
                       // $value->makeHidden('deleted_at');
                        $tag->makeHidden('created_at');
                        $tag->makeHidden('updated_at');
                    }
    
                }
                # code...
                $value->makeHidden('category');
              
                $value->makeHidden('deleted_at');
                $value->makeHidden('created_at');
                $value->makeHidden('updated_at');
            }
        }
        return $this->respondWithData($post);
       
    }

    

    public function updatePost(Request $request,$id)
    {
     
        $validator = Validator::make($request->all(), [
            'title' => 'required|string',
            'content' => 'required|string',
            'image' => 'image|mimes:jpeg,png,jpg,gif,svg|max:2048',
            'tag_id' => 'array',
        ]);
    
        if ($validator->fails()) {
            return response()->json(['errors' => $validator->errors()], 422);
        }
    
        $userId = Auth::check() ? Auth::id() : null;
        $imageArr = array();
    
        if (isset($request->tag_id)) {
            $imageArr = $request->tag_id;
            unset($request->tag_id);
        }
        $post = Post::find($id);
        if (!$post) {
            return response()->json(['error' => 'Post not found'], 404);
        }
    
        $post->user_id = $userId;
        $post->title = $request->title;
        $post->content = $request->content;
        $post->slug = Str::slug($request->title);
        $post->category_id = $request->category_id;
    
        if ($request->hasFile('image')) {
            $image = $request->file('image');
            $name = time() . '.' . $image->getClientOriginalExtension();
            $destinationPath = public_path('/images');
            $image->move($destinationPath, $name);
            $post->image = 'images/' . $name;
        }
    
        $post->save();
    
        if ($request->has('tag_id')) {
            $tag_ids = is_array($request->input('tag_id')) ? $request->input('tag_id') : [$request->input('tag_id')];
            $post->tags()->sync($tag_ids);
        }
    
        return response()->json(['message' => 'Post updated successfully', 'post' => $post]);
    }
    

    public function DeletePost($id)
    {
        $post = Post::find($id);
        if (!$post) {
            return response()->json(['error' => 'Post not found'], 404);
        }
        $post->delete();
        return response()->json(['message' => 'Post  deleted successfully']);
    }
}
<?php

namespace App\Http\Controllers;

use App\Models\Category;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class CategoryController extends ApiController
{
    /**
     * Store a newly created resource in storage.
     */
    public function storeCategory(Request $request)
    {
        
        $rules = [
            'name' => 'required',
        ];

        $data = $request->all();

        $validator = Validator::make($data, $rules);
        if ($validator->fails()) {
            $errors = $validator->errors()->all();
            return $this->respondWithError($errors);
        }

        Category::create($data);

        return $this->respondWithSuccess("Category added successfully!");
    }

    public function getCategory(Request $request)
    {
        $data = Category::Active()->get();
        if ($data) {

            $data->makeHidden('status');
            $data->makeHidden('slug');
            $data->makeHidden('created_at');
            $data->makeHidden('updated_at');
            return $this->respondWithData($data);
        } else {
            return $this->respondWithError(['Category Not Found']);
        }
    }

    // Other controller methods...
}
