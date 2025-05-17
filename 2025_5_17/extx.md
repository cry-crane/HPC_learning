路径：`/share/org/PARATERA/MCC20258768/zrh/cesm1.2.1/cesm.model/models/atm/cam/src/advection/slt/extx.F90`  

使用MPI进行进程间通信，通过MPI_SENDERCV实现了数据的双向交换，保持了周期性边界条件，每个进程只处理自己的子区域，通过与相邻进程交换边界数据来填充扩展区域。  
修改前代码：  

```f90

subroutine extx (pkcnst, pkdim, fb, kloop)

!----------------------------------------------------------------------- 
! 
! Purpose: 
! Copy data to the longitude extensions of the extended array
! 
! Method: 
! 
! Author: J. Olson
! 
!-----------------------------------------------------------------------
!
! $Id$
! $Author$
!
!-----------------------------------------------------------------------

  use shr_kind_mod, only: r8 => shr_kind_r8
  use scanslt,      only: plond, beglatex, endlatex, nxpt, nlonex
  implicit none

!------------------------------Arguments--------------------------------
  integer , intent(in) :: pkcnst    ! dimension construct for 3-D arrays
  integer , intent(in) :: pkdim     ! vertical dimension
  real(r8), intent(inout) :: fb(plond,pkdim*pkcnst,beglatex:endlatex) ! constituents
  integer,  intent(in) :: kloop ! Limit extent of loop of pkcnst
!-----------------------------------------------------------------------

!---------------------------Local variables-----------------------------
  integer i                 ! longitude index
  integer j                 ! latitude  index
  integer k                 ! vertical  index
  integer nlond             ! extended longitude dim
  integer i2pi              ! start of eastern long. extension
  integer pk                ! k extent to loop over
!-----------------------------------------------------------------------
!
! Fill west edge points.
!
  pk = pkdim*kloop
  if(nxpt >= 1) then
     do j=beglatex,endlatex
        do i=1,nxpt
           do k=1,pk
              fb(i,k,j) = fb(i+nlonex(j),k,j)
           end do
        end do
     end do
  end if
!
! Fill east edge points
!
  do j=beglatex,endlatex
     i2pi = nxpt + nlonex(j) + 1
     nlond = nlonex(j) + 1 + 2*nxpt
     do i=i2pi,nlond
        do k=1,pk
           fb(i,k,j) = fb(i-nlonex(j),k,j)
        end do
     end do
  end do

  return
end subroutine extx

```

修改后代码：

```f90
subroutine extx (pkcnst, pkdim, fb, kloop)

!----------------------------------------------------------------------- 
! 
! Purpose: 
! Copy data to the longitude extensions of the extended array
! 
! Method: 
! Using MPI for parallel data exchange
! 
! Author: J. Olson (Modified for MPI by Doubao)
! 
!-----------------------------------------------------------------------
!
! $Id$
! $Author$
!
!-----------------------------------------------------------------------

  use shr_kind_mod, only: r8 => shr_kind_r8
  use scanslt,      only: plond, beglatex, endlatex, nxpt, nlonex
  use mpi
  implicit none

!------------------------------Arguments--------------------------------
  integer , intent(in) :: pkcnst    ! dimension construct for 3-D arrays
  integer , intent(in) :: pkdim     ! vertical dimension
  real(r8), intent(inout) :: fb(plond,pkdim*pkcnst,beglatex:endlatex) ! constituents
  integer,  intent(in) :: kloop ! Limit extent of loop of pkcnst
!-----------------------------------------------------------------------

!---------------------------Local variables-----------------------------
  integer i                 ! longitude index
  integer j                 ! latitude  index
  integer k                 ! vertical  index
  integer nlond             ! extended longitude dim
  integer i2pi              ! start of eastern long. extension
  integer pk                ! k extent to loop over
  integer ierr              ! MPI error code
  integer my_rank           ! My process rank
  integer num_procs         ! Total number of processes
  integer left_rank, right_rank ! Ranks of left and right neighbors
  integer status(MPI_STATUS_SIZE) ! MPI status variable
  integer :: tag = 123       ! MPI tag for messages
  
  ! Buffers for sending and receiving halo regions
  real(r8), allocatable :: send_buf_west(:,:), recv_buf_west(:,:)
  real(r8), allocatable :: send_buf_east(:,:), recv_buf_east(:,:)
  integer :: west_size, east_size ! Sizes of halo regions
!-----------------------------------------------------------------------

  ! Initialize MPI variables
  call MPI_COMM_RANK(MPI_COMM_WORLD, my_rank, ierr)
  call MPI_COMM_SIZE(MPI_COMM_WORLD, num_procs, ierr)
  
  ! Determine neighbor ranks with periodic boundary conditions
  left_rank = mod(my_rank - 1, num_procs)
  right_rank = mod(my_rank + 1, num_procs)

  pk = pkdim*kloop
  west_size = nxpt * pk
  east_size = nxpt * pk
  
  ! Allocate buffers for halo exchange
  allocate(send_buf_west(nxpt, pk), recv_buf_west(nxpt, pk), &
           send_buf_east(nxpt, pk), recv_buf_east(nxpt, pk), stat=ierr)
  if (ierr /= 0) then
     print *, "Error allocating memory on rank ", my_rank
     call MPI_ABORT(MPI_COMM_WORLD, -1, ierr)
  end if

  ! Fill west edge points using MPI communication
  if(nxpt >= 1) then
     ! Prepare data to send to the left neighbor (western boundary)
     do j=beglatex,endlatex
        do k=1,pk
           do i=1,nxpt
              send_buf_west(i,k) = fb(i+nlonex(j),k,j)
           end do
        end do
        
        ! Send west data to left neighbor and receive from right neighbor
        call MPI_SENDRECV(send_buf_west, west_size, MPI_REAL8, left_rank, tag, &
                         recv_buf_west, west_size, MPI_REAL8, right_rank, tag, &
                         MPI_COMM_WORLD, status, ierr)
        
        ! Fill west edge points with received data
        do k=1,pk
           do i=1,nxpt
              fb(i,k,j) = recv_buf_west(i,k)
           end do
        end do
     end do
  end if

  ! Fill east edge points using MPI communication
  do j=beglatex,endlatex
     i2pi = nxpt + nlonex(j) + 1
     nlond = nlonex(j) + 1 + 2*nxpt
     
     ! Prepare data to send to the right neighbor (eastern boundary)
     do k=1,pk
        do i=i2pi,nlond
           send_buf_east(i-i2pi+1,k) = fb(i-nlonex(j),k,j)
        end do
     end do
     
     ! Send east data to right neighbor and receive from left neighbor
     call MPI_SENDRECV(send_buf_east, east_size, MPI_REAL8, right_rank, tag, &
                      recv_buf_east, east_size, MPI_REAL8, left_rank, tag, &
                      MPI_COMM_WORLD, status, ierr)
     
     ! Fill east edge points with received data
     do k=1,pk
        do i=i2pi,nlond
           fb(i,k,j) = recv_buf_east(i-i2pi+1,k)
        end do
     end do
  end do

  ! Cleanup
  deallocate(send_buf_west, recv_buf_west, send_buf_east, recv_buf_east, stat=ierr)

  return
end subroutine extx
```

运行结果：**754.867秒**，优化了3秒

