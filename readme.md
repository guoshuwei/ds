1.用户注册，存在推荐人，则该推荐人获得10积分（可配置修改）.如果推荐人有上一级，则推荐人的上一级推荐人获得5积分（可配置修改）
<?php

namespace App\Http\Controllers;
use Validator;
use Illuminate\Http\Request;

use Illuminate\Support\Facades\Redis;
use Hash;
use Illuminate\Support\Facades\Session;

class AuthController extends Controller
{

    public function __construct()
    {

    }


    // 在用户注册成功后调用该函数
    function updateRecommendPoints($userId, $recommendId, $level = 1) {
        // 更新推荐人积分
        $recommend = User::find($recommendId);
        if ($recommend) {
            $recommend->points += 10;
            $recommend->save();
        }

        // 递归向上更新上级推荐人积分
        if ($recommend && $level <= 2 && $recommend->recommend_id) {
            updateRecommendPoints($userId, $recommend->recommend_id, $level + 1);
        }
    }

    // 在注册控制器中调用该函数
    public function register(Request $request) {
        // 获取用户输入
        $data = $request->all();

        // 创建用户
        $user = new User();
        $user->name = $data['name'];
        $user->email = $data['email'];
        $user->password = bcrypt($data['password']);
        $user->save();

        // 更新推荐人积分
        if ($data['recommend_id']) {
            updateRecommendPoints($user->id, $data['recommend_id']);
        }

        // 返回注册成功的信息
        return response()->json(['message' => '注册成功']);
    }

}


2.使用填充文件生成一批用户，存在父子级关系，展示某用户的无限极子用户
php artisan make:seeder UsersTableSeeder命令来创建UsersTableSeeder.php文件
public function run()
{
    // 创建5个父级用户
    $parents = factory(App\User::class, 5)->create();

    // 为每个父级用户创建10个子用户
    foreach ($parents as $parent) {
        factory(App\User::class, 10)->create([
            'parent_id' => $parent->id
        ]);
    }
}
查找无限极子用户：
function getSubUsers($user, &$subUsers = []) {
    // 查找所有子用户
    $users = App\User::where('parent_id', $user->id)->get();

    // 将子用户添加到数组中
    foreach ($users as $subUser) {
        $subUsers[] = $subUser;
        getSubUsers($subUser, $subUsers);
    }

    return $subUsers;
}
$user = App\User::find(1);
$subUsers = getSubUsers($user);


3.使用定时任务每周一查询上一周用户的增长最大的前10个用户，写入日志文件中

1.生成计划任务：
php artisan make:command WeeklyTopUsers --command=weekly:top-users
2.实现任务调度
<?php
namespace App\Console\Commands;

use Illuminate\Console\GeneratorCommand;

class WeeklyTopUsers extends GeneratorCommand{

    public function handle()
    {
        $lastWeekStart = Carbon::now()->subWeek()->startOfWeek();
        $lastWeekEnd = Carbon::now()->subWeek()->endOfWeek();
        $topUsers = User::whereBetween('created_at', [$lastWeekStart, $lastWeekEnd])
            ->orderBy('created_at', 'desc')
            ->limit(10)
            ->get();

        $logData = 'Top 10 Users of Last Week:' . PHP_EOL;
        foreach ($topUsers as $user) {
            $logData .= sprintf('%s (%s)%s', $user->name, $user->email, PHP_EOL);
        }

        \Illuminate\Support\Facades\Log::info($logData);
    }
}
3.添加定时任务调度
protected function schedule(Schedule $schedule)
{
    $schedule->command('weekly:top-users')->mondays()->at('9:00');
}
4.启用定时任务
php artisan schedule:run >> /dev/null 2>&1
